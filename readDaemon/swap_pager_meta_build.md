### swp_pager_meta_build を読んでみる

`swp_pager_meta_build`はスワップページャのputpages関数で2回呼ばれています。

> File sys/vm/swap_pager.c, line 1219 ~  
オブジェクトの型がOBJT_SWAPじゃない時
```C
1219	if (object->type != OBJT_SWAP)
1220		swp_pager_meta_build(object, 0, SWAPBLK_NONE);
```


***

> line 1324 ~  
スワップブロックが確保できた時に、1ページ毎に呼ぶ
```C
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
```

何が行われているのでしょうか。

　

> File sys/vm/swap_pager.c, line 1800 ~
```C
static void
swp_pager_meta_build(vm_object_t object, vm_pindex_t pindex, daddr_t swapblk)
{
1803	struct swblock *swap;
1804	struct swblock **pswap;
1805	int idx;
```

引数は以下の意味のようです。
* `object` : 対象ページが属するオブジェクト
* `pindex` : ページのオブジェクト内のオフセット位置（vm_page構造体の説明より）
* `swapblk` : ページに対応するスワップブロックの位置

　

> line 1808 ~
```C
1808	/*
1809	 * Convert default object to swap object if necessary
1810	 */
1811	if (object->type != OBJT_SWAP) {
1812		object->type = OBJT_SWAP;
1813		object->un_pager.swp.swp_bcount = 0;
1814
1815		mtx_lock(&sw_alloc_mtx);
1816		if (object->handle != NULL) {
1817			TAILQ_INSERT_TAIL(
1818			    NOBJLIST(object->handle),
1819			    object, 
1820			    pager_object_list
1821			);
1822		} else {
1823			TAILQ_INSERT_TAIL(
1824			    &swap_pager_un_object_list,
1825			    object, 
1826			    pager_object_list
1827			);
1828		}
1829		mtx_unlock(&sw_alloc_mtx);
1830	}
```

コメントに書いてますが、デフォルトオブジェクトをスワップオブジェクトに変更するための処理です。  
本に書いてあった、デフォルトページャがスワップページャに書き換わるところです。  
ページャの呼出しは`object->type`がインデックスになる配列で管理されているので、`object->type`の書き換えがページャの変更になります。

　

> line 1837 ~
```C
retry:
1838	mtx_lock(&swhash_mtx);
1839	pswap = swp_pager_hash(object, pindex);
1840
1841	if ((swap = *pswap) == NULL) {
1842		int i;
1843
1844		if (swapblk == SWAPBLK_NONE)
1845			goto done;
1846
1847		swap = *pswap = uma_zalloc(swap_zone, M_NOWAIT);
1848		if (swap == NULL) {
1849			mtx_unlock(&swhash_mtx);
1850			VM_OBJECT_UNLOCK(object);
1851			VM_WAIT;
1852			VM_OBJECT_LOCK(object);
1853			goto retry;
1854		}
1855
1856		swap->swb_hnext = NULL;
1857		swap->swb_object = object;
1858		swap->swb_index = pindex & ~(vm_pindex_t)SWAP_META_MASK;
1859		swap->swb_count = 0;
1860
1861		++object->un_pager.swp.swp_bcount;
1862
1863		for (i = 0; i < SWAP_META_PAGES; ++i)
1864			swap->swb_pages[i] = SWAPBLK_NONE;
1865	}
```

* `1839:`スワップブロックを管理する構造体(struct swblock型)を、大域的なハッシュテーブルから取得します。  
objectとpindexがハッシュのキーになります。
* `1841:`戻り値がNULLを指すポインタだった場合、キーに対応するswblockが未だ無いので、作成します。
* `1858:``SWAP_META_MASK`は31(0x1F)です。この構造体はオブジェクト内の連続する32個のスワップブロックを管理するので、下5bitのindex情報はクリアして格納しています。
* `1864:``SWAP_META_PAGES`は管理するスワップブロック(ページ)の数で32です。スワップブロック情報はSWAPBLK_NONEで初期化します。

　

> line 1867 ~
```C
1867	/*
1868	 * Delete prior contents of metadata
1869	 */
1870	idx = pindex & SWAP_META_MASK;
1871
1872	if (swap->swb_pages[idx] != SWAPBLK_NONE) {
1873		swp_pager_freeswapspace(swap->swb_pages[idx], 1);
1874		--swap->swb_count;
1875	}
```

* `1870:`先ほどとは逆で下位5bitの情報をpindexから取り出し、構造体内でのindexとして使います。
* `1872:`指定された場所にすでに以前のスワップブロックが有った場合、解放します。

　

> line 1877 ~
```C
1877	/*
1878	 * Enter block into metadata
1879	 */
1880	swap->swb_pages[idx] = swapblk;
1881	if (swapblk != SWAPBLK_NONE)
1882		++swap->swb_count;
done:
1884	mtx_unlock(&swhash_mtx);
}
```

* '1880:'指定された場所に、ブロック位置情報を代入します。

　

#### まとめ
* スワップページャ（デフォルトページャ）のページアウト時の関数が呼ばれる度に、対象ページのオブジェクトを確認して、デフォルトオブジェクトならスワップオブジェクトに変更している。
* スワップブロックを管理する構造体(以降、swblock)は、オブジェクトとオブジェクト内のオフセットをキーにして、ハッシュテーブルから得られる。
* swblockは、必要に応じてメモリが割り当てられて作成される。
* swblockは32個のスワップブロックを管理し、1個のスワップブロックが1個のページに対応する。

