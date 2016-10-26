---
layout: post
title:  "Bugs of Android aapt remove function"
date:   2016-10-28
categories: android
tags: android aapt
---

使用 android aapt remove 功能的时候需要小心，存在两个比较严重的 bug。

aapt 相关的代码在 frameworks/base/tools/aapt 目录下，删除实现在 ZipFile.cpp 文件中。

### 1. 使用 aapt r 删除 apk（zip）文件中第一个 entry 的时候会失败。 ###    

这种情况下只会把 entry 从 central directory 区域删除，但是无法从 local file header 区域删除实际的文件内容。   
看一下这个bug的相关代码：
    
    status_t ZipFile::crunchArchive(void)
    {
        status_t result = NO_ERROR;
        int i, count;
        long delCount, adjust;

        /*
         * Roll through the set of files, shifting them as appropriate.  We
         * could probably get a slight performance improvement by sliding
         * multiple files down at once (because we could use larger reads
         * when operating on batches of small files), but it's not that useful.
         */
        count = mEntries.size();
        delCount = adjust = 0;
        for (i = 0; i < count; i++) {
            ZipEntry* pEntry = mEntries[i];
            long span;
            
            // !!! bug  这里是一个大bug，要删除的第一个第一个 entry 的 lfh offset 基本大多数情况为 0
            if (pEntry->getLFHOffset() != 0) { 
                long nextOffset;

                /* Get the length of this entry by finding the offset
                 * of the next entry.  Directory entries don't have
                 * file offsets, so we need to find the next non-directory
                 * entry.
                 */
                nextOffset = 0;
                for (int ii = i+1; nextOffset == 0 && ii < count; ii++)
                    nextOffset = mEntries[ii]->getLFHOffset();
                if (nextOffset == 0)
                    nextOffset = mEOCD.mCentralDirOffset;
                span = nextOffset - pEntry->getLFHOffset();

                assert(span >= ZipEntry::LocalFileHeader::kLFHLen);
            } else {
                // !!! 这里的注释写的很业余，可能他没弄明白zip文件格式，怎么可能lfh offset 为0 的就认为是目录呢？
                // 在 zip 文件格式中 目录 和 文件在这个 lhf offset上是没有区别的， 目录也是个正常的 entry 也有合法的 offset
                /* This is a directory entry.  It doesn't have           
                 * any actual file contents, so there's no need to
                 * move anything.
                 */
                span = 0;
            }


            if (pEntry->getDeleted()) {
                adjust += span;
                delCount++;

                delete pEntry;
                mEntries.removeAt(i);

                /* adjust loop control */
                count--;
                i--;
            } else if (span != 0 && adjust > 0) {
                /* shuffle this entry back */
                //printf("+++ Shuffling '%s' back %ld\n",
                //    pEntry->getFileName(), adjust);
                result = filemove(mZipFp, pEntry->getLFHOffset() - adjust,
                            pEntry->getLFHOffset(), span);
                if (result != NO_ERROR) {
                    /* this is why you use a temp file */
                    ALOGE("error during crunch - archive is toast\n");
                    return result;
                }

                pEntry->setLFHOffset(pEntry->getLFHOffset() - adjust);
            }
        }

        /*
         * Fix EOCD info.  We have to wait until the end to do some of this
         * because we use mCentralDirOffset to determine "span" for the
         * last entry.
         */
        mEOCD.mCentralDirOffset -= adjust;
        mEOCD.mNumEntries -= delCount;
        mEOCD.mTotalNumEntries -= delCount;
        mEOCD.mCentralDirSize = 0;  // mark invalid; set by flush()

        assert(mEOCD.mNumEntries == mEOCD.mTotalNumEntries);
        assert(mEOCD.mNumEntries == count);

        return result;
    }


### 2. 如果 central directory 和 local file header 区域的 entry index 顺序不一致，则会导致删除失败 ###

zip 文件格式中 central directory 表述的是zip文件中 n 个文件的描述， local file header 中存储的是真正的压缩数据。    
当真正要删除一个文件的时候，需要把 entry 从这两个地方都要删除。   
大多数情况下这两个区域的entry的排序是一致的。但是根据zip文件格式，并没有要求这样。因为 central directory 中的 entry 有个
offset 属性，指向 local file header 在文件中的 offset，用于寻找。所以没必要顺序一致。    
android gradle plugin 2.2 编译 debug 包就输出的 apk 就属于这个情况，两个区域的 entry 排序不一致。   

但是这个 ZipFile.cpp 的 remove 实现是假设两个区域的顺序是一样的，实现的时候只是把要删除的entry后的文件内容做一个偏移，
移动到要删除的 entry 的位置，所以如果两个区域顺序不一致，则会导致严重的删除问题，会破坏文件结构，输出的文件是个非法的zip文件。

这个问题修复也很好修复，在输出文件的时候，对所有的 entry 根据 offset 做个排序，然后再删除，然后输出即可。


具体的修复代码以及二进制文件请参考 [https://github.com/linghaolu/aapt](https://github.com/linghaolu/aapt)

当然是用 linux zip delete 功能也是 ok 的，但是会修改 zip 中的一些属性。不影响使用。

