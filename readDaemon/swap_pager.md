# スワップページャのソースを読んでみる

BSDカーネルの設計と実装 読書会 (12) で担当しました、"5.10.4 スワップページャ"について疑問点等あったので該当ソースコードを読んでみました。  
対象コードはFreeBSD 5.2-R です

　

218頁23行目付近
> そのため、新しいページャでは、それぞれのオブジェクトに割り当てられていた古い固定長のブロックマップを、確保されたスワップブロックの集合に対して構造体を割り当てる方式に変更した。それぞれの構造体は、最大32個のスワップブロックを管理することができる。

に対して、スワップブロックのサイズはいくつなのか良く分かりませんでした！

　

スワップブロックの確保が行われる、ページアウト要求時のスワップページャ操作の関数を読みます。
> File sys/vm/swap_pager.c, line 1197 ~
```C
void
swap_pager_putpages(vm_object_t object, vm_page_t *m, int count,
    boolean_t sync, int *rtvals)
{
```

各引数は以下の意味と思われます。
* `object` : 対象ページが属するオブジェクト
* `m` : ページアウト対象ページの配列
* `count` : mのページ数
* `sync` : 同期処理の指定
* `rtvals` : ページ毎の処理結果

　

> line 1219 ~
```C
1219	if (object->type != OBJT_SWAP)
1220		swp_pager_meta_build(object, 0, SWAPBLK_NONE);
```

最初にオブジェクトの型を調べて、`OBJT_SWAP`でない場合は`swp_pager_meta_build`をコールします。  
この関数はまた後で出てきます。

　

> line 1269 ~
```C
1269	for (i = 0; i < count; i += n) {
1270		int s;
1271		int j;
1272		struct buf *bp;
1273		daddr_t blk;
1274
1275		/*
1276		 * Maximum I/O size is limited by a number of factors.
1277		 */
1278		n = min(BLIST_MAX_ALLOC, count - i);
1279		n = min(n, nsw_cluster_max);
...
1289		while (
1290		    (blk = swp_pager_getswapspace(n)) == SWAPBLK_NONE &&
1291		    n > 4
1292		) {
1293			n >>= 1;
1294		}
1295		if (blk == SWAPBLK_NONE) {
1296			for (j = 0; j < n; ++j)
1297				rtvals[i+j] = VM_PAGER_FAIL;
1298			splx(s);
1299			continue;
1300		}
```

ループを回して、ページアウトするページ数分(`count`)のブロックを`swp_pager_getswapspace(n)`で確保します。
* `1278:`nは1回のgetで要求するページ数で、残りの必要な数(`count-i`)と各種MAX値を越えない範囲のminで決定します。
* `1289:``swp_pager_getswapspace`を呼出し、失敗時(`SWAPBLK_NONE`)はnを右シフトで減算してリトライします。  
成功時はblkは獲得したスワップブロックの位置が返ります。
* `1295:`nが4以下でも失敗した時はwhileを抜けて、`rtvals`の該当する箇所を`VM_PAGER_FAIL`で埋めて、ループの最初に戻ります。  

**というわけでスワップブロックはページアウトするページと同じ数だけ要求しています。**  
あと、できるだけまとまった数を一度に確保しようとします。  
スワップブロックを管理する構造体は未だ出てきません。

　

> line 1306 ~  /* まだforループ内です */
```C
1306		if (sync == TRUE) {
1307			bp = getpbuf(&nsw_wcount_sync);
1308		} else {
1309			bp = getpbuf(&nsw_wcount_async);
1310			bp->b_flags = B_ASYNC;
1311		}
1312		bp->b_flags |= B_PAGING;
1313		bp->b_iocmd = BIO_WRITE;
1314
1315		pmap_qenter((vm_offset_t)bp->b_data, &m[i], n);
1316
1317		bp->b_rcred = crhold(thread0.td_ucred);
1318		bp->b_wcred = crhold(thread0.td_ucred);
1319		bp->b_bcount = PAGE_SIZE * n;
1320		bp->b_bufsize = PAGE_SIZE * n;
1321		bp->b_blkno = blk;
1322
1323		VM_OBJECT_LOCK(object);
1324		for (j = 0; j < n; ++j) {
1325			vm_page_t mreq = m[i+j];
1326
1327			swp_pager_meta_build(
1328			    mreq->object, 
1329			    mreq->pindex,
1330			    blk + j
1331			);
1332			vm_page_dirty(mreq);
1333			rtvals[i+j] = VM_PAGER_OK;
1334
1335			vm_page_lock_queues();
1336			vm_page_flag_set(mreq, PG_SWAPINPROG);
1337			vm_page_unlock_queues();
1338			bp->b_pages[j] = mreq;
1339		}
1340		VM_OBJECT_UNLOCK(object);
1341		bp->b_npages = n;
```

バッファを確保して、ディスク書込みの準備をします。
* `1306:`同期、非同期処理のどちらの場合も`getpbuf`でディスク書込みするためのバッファ(physical buffer)を獲得します。  
その後バッファにIO方向(`BIO_WRITE`)、IOの資格情報、書込みサイズ(`PAGE_SIZE * n`)、書込み先(`blk`)の情報を格納します。  
（ここの書込みサイズからも、スワップブロック1つ分のサイズはPAGE_SIZEと同じな気がします...）
* `1324:`nは確保できたスワップブロックのサイズですが、1個ずつ`swp_pager_meta_build`を呼びます。  
引数は各ブロックに対応するページのオブジェクトとインデックス、ページに対応するスワップブロック位置です。  
この関数の先で各ページと対応するスワップブロックの管理をします。

この後、バッファ(bp）を使って同期または非同期の書込みを行います。全ページ分終了するとスワップページャの関数も終わります。

　

### まとめ
* スワップブロックはページアウトするページと同じ数だけ確保される。
* できるだけ一度にまとめて確保しようとする。
* スワップブロックの位置が確定した後、バッファ構造体にIO情報をまとめて、スワップ領域への書込みを行う。
* (スワップブロックはページと1対1なので、ページと同じサイズだと思いますが...、 **確定できるほどの情報は得られませんでした！**)

　

途中で呼び出されていた関数、[swap_pager_meta_buildはこちら](https://github.com/kusabanachi/notes/blob/master/readDaemon/swap_pager_meta_build.md)

