---
search:
   keywords: ['storage', 'PLocal', 'write ahead log', 'WAL', 'journal']
---

# PLocal WAL (Journal)

Write Ahead Log, **WAL** hereafter, is operation log which is used to store data about operations which are performed on disk cache page. WAL is enabled by default.

You could disable the journal (WAL) for some operations where reliability is not necessary:

    -Dstorage.useWAL=false

By default, the WAL files are written in the database folder. Since these files can grow very fast, it's a best practice to store it in a dedicated partition. WAL files are written in append-only mode, so there is not much difference on using a SSD or a normal HDD. If you have a SSD we suggest to use it for database files only, not WAL.

To setup a different location than database folder, set the `WAL_LOCATION`variable.

```java
OGlobalConfiguration.WAL_LOCATION.setValue("/temp/wal")
```

or at JVM level:
```
java ... -Dstorage.wal.path=/temp/wal ...
```

This log is not a high-level log, which on other hand is used to log operations on record level. For each page change, the following values are stored:

1. offset and length of the chunk of bytes which was changed.
2. previous value of the chunk of bytes.
3. replaced (new) value of the chunk of bytes.

As you can see WAL contains not logical but raw (in form of the chunk of bytes) presentation of data which was/is contained inside of the page. Such format of the record of write-ahead log allows to apply the same changes to the page several times and as result allows do not flush cache content after each TX operation but do such flush on demand and flush only chosen pages instead of the whole cache. The second advantage is following if storage is crashed during data restore operation it can be restored repeatedly.

Let's say we have a page where following changes are done.

1. 10 bytes at the beginning were changed.
2. 10 bytes at the end were changed.

Storage is crashed during the middle of page flush, which does not mean that first 10 bytes are written, so let's suppose that the last 10 changed byte was written, but first 10 bytes were not.

During data restore we apply all operations stored in WAL one by one, which means that we set first 10 bytes of the changed page and then last 10 bytes of this page. So the changed page will have correct state does not matter whether its state was flushed to the disk or not.

WAL file is split into pages and segments, each page contains in header CRC32 code of page content and "magic number".
When operation records are logged to WAL they are serialized and binary content appended to the current page if it is not enough space left in the page to accommodate binary presentation of the whole record, the part of the binary content (which does not fit inside of current page) will be put inside of next record. It is important to avoid gaps (free space) inside of pages. As with any other files, WAL can be corrupted because of power failure and detection of gaps inside WAL pages is one of the approaches how database separates broken and "healthy" WAL pages. More about this later.

Any operation may include not single but several pages, to avoid data inconsistency all operations on several records inside of one logical operation are considered as single atomic operation.
To achieve this functionality following types of WAL records were introduced:

1. atomic operation start.
2. atomic operation end.
3. record which contains changes is done in a single page inside of atomic operation.

These records contain following fields:

1. Atomic operation start record contains following fields:
   1. Atomic operation id (uuid).
   2. LSN (log sequence number) - physical position of log record inside WAL.
2. Atomic operation end record contains following fields:
   1. Atomic operation id (uuid).
   2. LSN (log sequence number) - physical position of log record inside WAL.
   3. rollback flag  - indicates whether given atomic operation should be rolled back.
3. Record which contains page changes contains following fields:
   1. LSN (log sequence number) - physical position of log record inside WAL.
   2. page index and file id of changed page.
   3. Page changes itself.
   4. LSN of change which was applied to the current page before given one - prevLSN.

The last record's type (page changes container) contains the field (d. item) which deserves additional explanation. Each cache page contains following "system" fields:

1. CRC32 code of the rest of content.
2. magic number
3. LSN of last change applied to the page - page LSN.

Every time we perform changes on the page before we release it back to the cache we log page changes to the WAL, assign LSN of WAL record as the "page LSN" and only after that release page back to the cache.

When WAL flushes it's pages it does not do it at once when the current page is filled it is put in the cache and is flushed in the background along with other cached pages. Flush is performed every second in a background thread (it is a trade-off between performance and durability). But there are two exceptions when flush is performed in the thread which put the record in WAL:

1. If WAL page's cache is exhausted.
2. If cache page is flushed, page LSN is compared with LSN of last flushed WAL record and if page LSN  is more than LSN of flushed WAL record then flush of WAL pages is triggered. LSN is physical position of WAL record, because of WAL is an append-only log so if "page LSN" is more than LSN of flushed record it means that changes for given page were logged but not flushed, but we can restore state of page only and only if all page changes will be contained in WAL too.

Given all of this data restore process looks like following:
<pre>
begin
go through all WAL records one by one
gather together all atomic operation records in one batch
when "atomic operation end" record was found
  if commit should be performed
    go through all atomic operation records from first to last, apply all page changes, set page LSN to the LSN of applied WAL record.
  else
    go through all atomic operation records from last to first, set old page's content, set page LSN to the WALRecord.prevLSN value.
  endif
end
</pre>

As it is written before WAL files are usual files and they can be flushed only partially if power is switched off during WAL cache flush. There are two cases of how WAL pages can be broken:

1. Pages are flushed partially.
2. Some of the pages are completely flushed, some are not flushed.

The first case is very easy to detect and resolve:

1. When we open WAL during DB start we verify that size of WAL multiplies of WAL page size if it is not WAL size is truncated to page size.
2. When we read pages one by one we verify CR32 and the magic number of each page. If a page is broken we stop data restore procedure here.

The second case is a bit more tricky. Because WAL is an append-only log, there are two possible sub-cases,
let's suppose we have 3 pages after 2nd (broken) flush. First and the first half of the second page were flushed during first flush and second half of the second page and the third page were flushed during the second flush.
Because the second flush was interrupted by power failure we can have two possible states:

1. The second half of page was flushed but third was not. It is easy to detect by checking CRC and magic number values.
2. The second half of page is not flushed but the third page is flushed. In such case, CRC and magic number values will be correct and we can not use them instead of this when we read WAL page we check if this page has free space if it has then we check if this is the last page if it is not we mark this WAL page as broken.


