---
layout: posts
title:  "QCoW2 Snapshot Breakdown"
author: Minjong Ha
published: false
date:   2022-10-18 16:59:00 +0900
---

## Introduction
I already wrote a post about VM image snapshot in ${url_link}.
However, there was a chance that analyzing more detail actions of qcow2 snapshot with source code.
In this post, I will explain how exactly qcow2 and qemu generate snapshot (overlay image), especially focusing on the copy-on-write.

## Qcow2 Architecture
I already explained the architecture of qcow2 image in ${url_link}.
To understand more details about I/O and snapshots, I analyzed qcow2 header and source codes based on qemu-7.0.0.

Qcow2 header contains several arguments for its action.
"nb_snapshots" and "snapshots_offset" represent the number of snapshot and offsets that image has.
"image_backing_file" represents the path of backing file that image references.

Since the describes about the internal and external snapshots in QEMU related documents<sup>[1](#footnote_1)</sup> and difference between qcow1 and qcow2<sup>[1](#footnote_2)</sup>, I misundertood that internal and external snapshots are added features in qcow2.
Actually, qcow1 already supports external snapshot; I think the actual name is backing file, and supports internal snapshot since qcow2.

Based on the name and source codes, I start to think that qemu does not consider the external snapshot as a snapshot.
In my opinion, its original name is backing file, and qemu support it from qcow (version 1).
Of course it can be used as a snapshot, however, it is more likely to use it as a template in oVirt.
oVirt supports template feature, which means that you can create a VM template based on the VM image.
Usually, the size of the template is thin-provisioned VM image.
If you generate new VM based on the template, new VM image only has a few MB size and grows.
It looks like that oVirt uses backing file feature to implement template.

From qcow2, qemu supports its new feature: snapshot (internal snapshot).
Unlike the backing file, internal snapshot contains every snapshot (overlay) architecture within the single file.
Qemu can switch the VM between the snapshots during the runtime of VM.
Moreover, it also provide memory state snapshot either, so it can reload the VM state when the snapshot created.

## Qcow2 Snapshot
In this section, I will describe more detail actions about snapshot with source codes.

```c
/* if no id is provided, a new one is constructed */
int qcow2_snapshot_create(BlockDriverState *bs, QEMUSnapshotInfo *sn_info)
{
    BDRVQcow2State *s = bs->opaque;
    QCowSnapshot *new_snapshot_list = NULL;
    QCowSnapshot *old_snapshot_list = NULL;
    QCowSnapshot sn1, *sn = &sn1;
    int i, ret;
    uint64_t *l1_table = NULL;
    int64_t l1_table_offset;

    if (s->nb_snapshots >= QCOW_MAX_SNAPSHOTS) {
        return -EFBIG;
    }

    if (has_data_file(bs)) {
        return -ENOTSUP;
    }

    memset(sn, 0, sizeof(*sn));

    /* Generate an ID */
    find_new_snapshot_id(bs, sn_info->id_str, sizeof(sn_info->id_str));

    /* Populate sn with passed data */
    sn->id_str = g_strdup(sn_info->id_str);
    sn->name = g_strdup(sn_info->name);

    sn->disk_size = bs->total_sectors * BDRV_SECTOR_SIZE;
    sn->vm_state_size = sn_info->vm_state_size;
    sn->date_sec = sn_info->date_sec;
    sn->date_nsec = sn_info->date_nsec;
    sn->vm_clock_nsec = sn_info->vm_clock_nsec;
    sn->icount = sn_info->icount;
    sn->extra_data_size = sizeof(QCowSnapshotExtraData);

    /* Allocate the L1 table of the snapshot and copy the current one there. */
    l1_table_offset = qcow2_alloc_clusters(bs, s->l1_size * L1E_SIZE);
    if (l1_table_offset < 0) {
        ret = l1_table_offset;
        goto fail;
    }
		sn->l1_table_offset = l1_table_offset;
    sn->l1_size = s->l1_size;

    l1_table = g_try_new(uint64_t, s->l1_size);
    if (s->l1_size && l1_table == NULL) {
        ret = -ENOMEM;
        goto fail;
    }

    for(i = 0; i < s->l1_size; i++) {
        l1_table[i] = cpu_to_be64(s->l1_table[i]);
    }

    ret = qcow2_pre_write_overlap_check(bs, 0, sn->l1_table_offset,
                                        s->l1_size * L1E_SIZE, false);
    if (ret < 0) {
        goto fail;
    }

    ret = bdrv_pwrite(bs->file, sn->l1_table_offset, l1_table,
                      s->l1_size * L1E_SIZE);
    if (ret < 0) {
        goto fail;
    }

    g_free(l1_table);
    l1_table = NULL;

    /*
     * Increase the refcounts of all clusters and make sure everything is
     * stable on disk before updating the snapshot table to contain a pointer
     * to the new L1 table.
     */
    ret = qcow2_update_snapshot_refcount(bs, s->l1_table_offset, s->l1_size, 1);
    if (ret < 0) {
        goto fail;
    }

    /* Append the new snapshot to the snapshot list */
    new_snapshot_list = g_new(QCowSnapshot, s->nb_snapshots + 1);
    if (s->snapshots) {
        memcpy(new_snapshot_list, s->snapshots,
               s->nb_snapshots * sizeof(QCowSnapshot));
        old_snapshot_list = s->snapshots;
    }
    s->snapshots = new_snapshot_list;
    s->snapshots[s->nb_snapshots++] = *sn;

    ret = qcow2_write_snapshots(bs);
    if (ret < 0) {
        g_free(s->snapshots);
        s->snapshots = old_snapshot_list;
				s->nb_snapshots--;
        goto fail;
    }

    g_free(old_snapshot_list);

    /* The VM state isn't needed any more in the active L1 table; in fact, it
     * hurts by causing expensive COW for the next snapshot. */
    qcow2_cluster_discard(bs, qcow2_vm_state_offset(s),
                          ROUND_UP(sn->vm_state_size, s->cluster_size),
                          QCOW2_DISCARD_NEVER, false);

#ifdef DEBUG_ALLOC
    {
      BdrvCheckResult result = {0};
      qcow2_check_refcounts(bs, &result, 0);
    }
#endif
    return 0;

fail:
    g_free(sn->id_str);
    g_free(sn->name);
    g_free(l1_table);

    return ret;
}
```

Above codes are located in qemu/block/qcow2-snapshot.c, and it represents the start point of snapshot.
When snapshot is created, L1 table is allocated by qcow2_alloc_clusters().
New L1 table in snapshot has same size with the L1 table of base image and also has same values.
It means that both L1 tables has same L2 table offsets.
Qcow2 snapshot does not allocate independent L2 table when it snapshots the image.


## References

## Footnotes

<a name='footnote_1'><1></a>: qcow2 presents two snapshot: internal and external (https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html)

<a name='footnote_2'><2></a>: The difference from the original version is that qcow2 supports multiple snapshots using a newer, more flexible model for storing them. (https://en.wikipedia.org/wiki/Qcow)
