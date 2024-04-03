# PATCH 1/5

Promotion and demotion related statistics can help better understand
the effectiveness of the page placement mechanism. we want to know
among the promoted/demoted pages what is the distribution of anon and
file pages. how much of the demoted pages become promotion candidate
can give us insight on whether the page placement mechanism is thrashing
among the NUMA nodes. we can also use this information to rate limit the
migration across the NUMA nodes.

Promotion can fail for many reasons, e.g., target node having low memory,
page refcount being abnormal, whole system being low on memory etc. Adding
counters to track the failure reasons will give the detailed info about
why and where it fails, and help debugging the system.

To track the demoted pages, PG\_demoted bit is introduced for pages that get demoted. Upon demotion, PG\_demoted bit is set in thepage flag. upon promotion, the bit gets reset for that page.

## promotion related statistics:

pgpromote\_candidate - candidates that get selected for promotion
pgpromote\_candidate\_demoted - promotion candidate that got demoted earlier
pgpromote\_candidate\_anon - promotion candidate that are anon
pgpromote\_candidate\_file - promotion candidate that are file
pgpromote\_tried - pages that had a try to migrate via NUMA Balancing
pgpromote\_file- successfully promoted file pages
pgpromote\_anon - successfully promoted anon pages

## promotion failure related statistics:

pgmigrate\_fail\_dst\_node\_full - failed as the target node is full
pgmigrate\_fail\_numa\_isolate - failed in isolating numa page
pgmigrate\_fail\_nomem - failed as no memory left in the system
pgmigrate\_fail\_refcount - failed as ref count mismatched

## demotion related statistics:

pgdemote\_file - successfully demoted file pages
pgdemote\_anon - successfully demoted anon pages

## 修改文件

include/linux/mempolicy\.h \| 4 \+\-
include/linux/page\-flags\.h \| 9 \+\+\+\+
include/linux/page\_ext\.h \| 3 \+\+
include/linux/sched/numa\_balancing\.h \| 63 \+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\-
include/linux/vm\_event\_item\.h \| 13 \+\+\+\+\+\+
include/trace/events/mmflags\.h \| 10 \+\+\+\+\-
kernel/sched/fair\.c \| 12 \+\+\+\+\+\-
kernel/sched/sched\.h \| 1 \+
mm/huge\_memory\.c \| 2 \+\-
mm/memory\.c \| 2 \+\-
mm/mempolicy\.c \| 7 \+\+\+\-
mm/migrate\.c \| 48 \+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\-\-\-\-\-
mm/vmscan\.c \| 8 \+\+\+\+
mm/vmstat\.c \| 13 \+\+\+\+\+\+
14 files changed, 174 insertions(+), 21 deletions(-)

## /include/linux/mempolicy.h
@@ -184,7 +184,7 @@ extern void mpol\_to\_str(char \*buffer, int maxlen, struct mempolicy *pol);*
/ Check if a vma is migratable \*/
extern bool vma\_migratable(struct vm\_area\_struct \*vma);

-extern int mpol\_misplaced(struct page \*, struct vm\_area\_struct \*, unsigned long);
+extern int mpol\_misplaced(struct page \*, struct vm\_area\_struct \*, unsigned long, int);
extern void mpol\_put\_task\_policy(struct task\_struct \*);

extern bool numa\_demotion\_enabled;
@@ -284,7 +284,7 @@ static inline int mpol\_parse\_str(char \*str, struct mempolicy \*\*mpol)
#endif

static inline int mpol\_misplaced(struct page \*page, struct vm\_area\_struct \*vma,

*
```
  		 unsigned long address)
```

*
```
  		 unsigned long address, int flags)
```

{
return -1; /\* no node preference \*/
}

## /include/linux/page-flags.h

@@ -137,6 +137,9 @@ enum pageflags {
#endif
#ifdef CONFIG\_64BIT
PG\_arch\_2,
+#ifdef CONFIG\_NUMA\_BALANCING

* PG\_demoted,
+#endif
#endif
\_\_NR\_PAGEFLAGS,

@@ -443,6 +446,12 @@ TESTCLEARFLAG(Young, young, PF\_ANY)
PAGEFLAG(Idle, idle, PF\_ANY)
#endif

+#if defined(CONFIG\_NUMA\_BALANCING) && defined(CONFIG\_64BIT)
+TESTPAGEFLAG(Demoted, demoted, PF\_NO\_TAIL)
+SETPAGEFLAG(Demoted, demoted, PF\_NO\_TAIL)
+TESTCLEARFLAG(Demoted, demoted, PF\_NO\_TAIL)
+#endif
+
/\*

* PageReported() is used to track reported free pages within the Buddy
* allocator. We can use the non-atomic version of the test and set

## /include/linux/page\_ext.h

diff --git a/include/linux/page\_ext.h b/include/linux/page\_ext.h
index aff81ba31bd8..1a1e632031d3 100644
\-\-\- a/include/linux/page\_ext\.h
+++ b/include/linux/page\_ext.h
@@ -23,6 +23,9 @@ enum page\_ext\_flags {
PAGE\_EXT\_YOUNG,
PAGE\_EXT\_IDLE,
#endif
+#if defined(CONFIG\_NUMA\_BALANCING) && !defined(CONFIG\_64BIT)

* PAGE\_EXT\_DEMOTED,
+#endif
};

/\*

## /include/linux/sched/numa\_balancing.h

@@ -8,12 +8,14 @@
\*/

#include \<linux/sched.h>
+#include \<linux/page-flags.h>

#define TNF\_MIGRATED	0x01
#define TNF\_NO\_GROUP	0x02
#define TNF\_SHARED	0x04
#define TNF\_FAULT\_LOCAL	0x08
#define TNF\_MIGRATE\_FAIL 0x10
+#define TNF\_DEMOTED	0x40

#ifdef CONFIG\_NUMA\_BALANCING
extern void task\_numa\_fault(int last\_node, int node, int pages, int flags);
@@ -21,7 +23,53 @@ extern pid\_t task\_numa\_group\_id(struct task\_struct \*p);
extern void set\_numabalancing\_state(bool enabled);
extern void task\_numa\_free(struct task\_struct \*p, bool final);
extern bool should\_numa\_migrate\_memory(struct task\_struct \*p, struct page \*page,

*
```
  			int src_nid, int dst_cpu);
```

*
```
  			int src_nid, int dst_cpu, int flags);
```

+#ifdef CONFIG\_64BIT
+static inline bool page\_is\_demoted(struct page \*page)
+{

* return PageDemoted(page);
+}
*

+static inline void set\_page\_demoted(struct page \*page)
+{

* SetPageDemoted(page);
+}
*

+static inline bool test\_and\_clear\_page\_demoted(struct page \*page)
+{

* return TestClearPageDemoted(page);
+}
+#else /\* !CONFIG\_64BIT \*/
+static inline bool page\_is\_demoted(struct page \*page)
+{
* struct page\_ext \*page\_ext = lookup\_page\_ext(page);
* 
* if (unlikely(!page\_ext))
*
```
  return false;
```

* 
* return test\_bit(PAGE\_EXT\_DEMOTED, &page\_ext->flags);
+}
*

+static inline void set\_page\_demoted(struct page \*page)
+{

* struct page\_ext \*page\_ext = lookup\_page\_ext(page);
* 
* if (unlikely(!page\_ext))
*
```
  return false;
```

* 
* return set\_bit(PAGE\_EXT\_DEMOTED, &page\_ext->flags);
+}
*

+static inline bool test\_and\_clear\_page\_demoted(struct page \*page)
+{

* struct page\_ext \*page\_ext = lookup\_page\_ext(page);
* 
* if (unlikely(!page\_ext))
*
```
  return false;
```

* 
* return test\_and\_clear\_bit(PAGE\_EXT\_DEMOTED, &page\_ext->flags);
+}
+#endif /\* !CONFIG\_64BIT \*/
#else
static inline void task\_numa\_fault(int last\_node, int node, int pages,
int flags)
@@ -38,10 +86,21 @@ static inline void task\_numa\_free(struct task\_struct \*p, bool final)
{
}
static inline bool should\_numa\_migrate\_memory(struct task\_struct \*p,

*
```
  		struct page *page, int src_nid, int dst_cpu)
```

*
```
  		struct page *page, int src_nid, int dst_cpu, int flags)
```

{
return true;
}
+static inline bool page\_is\_demoted(struct page \*page)
+{

* return false;
+}
+static inline void set\_page\_demoted(struct page \*page)
+{
+}
+static inline bool test\_and\_clear\_page\_demoted(struct page \*page)
+{
* return false;
+}
#endif

#endif /\* \_LINUX\_SCHED\_NUMA\_BALANCING\_H \*/
## /include/linux/vm\_event\_item.h文件
diff --git a/include/linux/vm\_event\_item.h b/include/linux/vm\_event\_item.h
index b136ed6224a2..9cb43a2998cb 100644
\-\-\- a/include/linux/vm\_event\_item\.h
+++ b/include/linux/vm\_event\_item.h
@@ -35,6 +35,8 @@ enum vm\_event\_item { PGPGIN, PGPGOUT, PSWPIN, PSWPOUT,
PGSTEAL\_DIRECT,
PGDEMOTE\_KSWAPD,
PGDEMOTE\_DIRECT,

*
```
  PGDEMOTE_FILE,
```

*
```
  PGDEMOTE_ANON,
  PGSCAN_KSWAPD,
  PGSCAN_DIRECT,
  PGSCAN_DIRECT_THROTTLE,
```

@@ -56,9 +58,20 @@ enum vm\_event\_item { PGPGIN, PGPGOUT, PSWPIN, PSWPOUT,
NUMA\_HINT\_FAULTS,
NUMA\_HINT\_FAULTS\_LOCAL,
NUMA\_PAGE\_MIGRATE,

*
```
  PGPROMOTE_CANDIDATE,		/* candidates get selected for promotion */
```

*
```
  PGPROMOTE_CANDIDATE_DEMOTED,/* promotion candidate that got demoted earlier */
```

*
```
  PGPROMOTE_CANDIDATE_ANON,	/* promotion candidate that are anon */
```

*
```
  PGPROMOTE_CANDIDATE_FILE,	/* promotion candidate that are file */
```

*
```
  PGPROMOTE_TRIED,			/* tried to migrate via NUMA balancing */
```

*
```
  PGPROMOTE_FILE,				/* successfully promoted file pages  */
```

*
```
  PGPROMOTE_ANON,				/* successfully promoted anon pages  */
```

#endif
#ifdef CONFIG\_MIGRATION
PGMIGRATE\_SUCCESS, PGMIGRATE\_FAIL,

*
```
  PGMIGRATE_DST_NODE_FULL_FAIL,	/* failed as the target node is full */
```

*
```
  PGMIGRATE_NUMA_ISOLATE_FAIL,	/* failed in isolating numa page */
```

*
```
  PGMIGRATE_NOMEM_FAIL,			/* failed as no memory left */
```

*
```
  PGMIGRATE_REFCOUNT_FAIL,		/* failed in ref count */
  THP_MIGRATION_SUCCESS,
  THP_MIGRATION_FAIL,
  THP_MIGRATION_SPLIT,
```

## /include/trace/events/mmflags.h文件
diff --git a/include/trace/events/mmflags.h b/include/trace/events/mmflags.h
index 67018d367b9f..7ba2c2702ef7 100644
\-\-\- a/include/trace/events/mmflags\.h
+++ b/include/trace/events/mmflags.h
@@ -85,6 +85,13 @@
#define IF\_HAVE\_PG\_ARCH\_2(flag,string)
#endif

+#if defined(CONFIG\_NUMA\_BALANCING) && defined(CONFIG\_64BIT)
+#define IF\_HAVE\_PG\_DEMOTED(flag, string) ,{1UL << flag, string}
+#else
+#define IF\_HAVE\_PG\_DEMOTED(flag, string)
+#endif
+
+
#define \_\_def\_pageflag\_names
{1UL << PG\_locked,		"locked"	},
{1UL << PG\_waiters,		"waiters"	},
@@ -112,7 +119,8 @@ IF\_HAVE\_PG\_UNCACHED(PG\_uncached,	"uncached"	)
IF\_HAVE\_PG\_HWPOISON(PG\_hwpoison,	"hwpoison"	)
IF\_HAVE\_PG\_IDLE(PG\_young,		"young"		)
IF\_HAVE\_PG\_IDLE(PG\_idle,		"idle"		)
-IF\_HAVE\_PG\_ARCH\_2(PG\_arch\_2,		"arch\_2"	)
+IF\_HAVE\_PG\_ARCH\_2(PG\_arch\_2,		"arch\_2")
+IF\_HAVE\_PG\_DEMOTED(PG\_demoted,		"demoted")

#define show\_page\_flags(flags)
\(flags\) ? \_\_print\_flags\(flags\, "\|"\,
diff --git a/kernel/sched/fair.c b/kernel/sched/fair.c
index 572f312cc803..210612c9d1e9 100644
\-\-\- a/kernel/sched/fair\.c
+++ b/kernel/sched/fair.c
@@ -1416,12 +1416,22 @@ static inline unsigned long group\_weight(struct task\_struct \*p, int nid,
}

bool should\_numa\_migrate\_memory(struct task\_struct \*p, struct page \* page,

*
```
  		int src_nid, int dst_cpu)
```

*
```
  		int src_nid, int dst_cpu, int flags)
```

{
struct numa\_group \*ng = deref\_curr\_numa\_group(p);
int dst\_nid = cpu\_to\_node(dst\_cpu);
int last\_cpupid, this\_cpupid;

* count\_vm\_numa\_event(PGPROMOTE\_CANDIDATE);
* 
* if (flags & TNF\_DEMOTED)
*
```
  count_vm_numa_event(PGPROMOTE_CANDIDATE_DEMOTED);
```

* 
* if (page\_is\_file\_lru(page))
*
```
  count_vm_numa_event(PGPROMOTE_CANDIDATE_FILE);
```

* else
*
```
  count_vm_numa_event(PGPROMOTE_CANDIDATE_ANON);
```

* this\_cpupid = cpu\_pid\_to\_cpupid(dst\_cpu, current->pid);
last\_cpupid = page\_cpupid\_xchg\_last(page, this\_cpupid);

## /kernel/sched/sched.h文件
diff --git a/kernel/sched/sched.h b/kernel/sched/sched.h
index eee49ce2d596..6057ad67d223 100644
\-\-\- a/kernel/sched/sched\.h
+++ b/kernel/sched/sched.h
@@ -51,6 +51,7 @@
#include \<linux/kthread.h>
#include \<linux/membarrier.h>
#include \<linux/migrate.h>
+#include <linux/mm\_inline.h>
#include <linux/mmu\_context.h>
#include \<linux/nmi.h>
#include <linux/proc\_fs.h>

## /mm/huge\_memory.c文件
diff --git a/mm/huge\_memory.c b/mm/huge\_memory.c
index bc642923e0c9..e9d7b9125c5e 100644
\-\-\- a/mm/huge\_memory\.c
+++ b/mm/huge\_memory.c
@@ -1475,7 +1475,7 @@ vm\_fault\_t do\_huge\_pmd\_numa\_page(struct vm\_fault \*vmf, pmd\_t pmd)
\* page\_table\_lock if at all possible
\*/
page\_locked = trylock\_page(page);

* target\_nid = mpol\_misplaced(page, vma, haddr);

* target\_nid = mpol\_misplaced(page, vma, haddr, flags);
if (target\_nid == NUMA\_NO\_NODE) {
/\* If the page was locked, there are no parallel migrations \*/
if (page\_locked)
diff --git a/mm/memory.c b/mm/memory.c
index c8083f571c89..314fe3b2f462 100644
\-\-\- a/mm/memory\.c
+++ b/mm/memory.c
@@ -4131,7 +4131,7 @@ static int numa\_migrate\_prep(struct page \*page, struct vm\_area\_struct \*vma,
\*flags \|= TNF\_FAULT\_LOCAL;
}

* return mpol\_misplaced(page, vma, addr);

* return mpol\_misplaced(page, vma, addr, \*flags);
}

static vm\_fault\_t do\_numa\_page(struct vm\_fault \*vmf)
## /mm/mempolicy.c文件
diff --git a/mm/mempolicy.c b/mm/mempolicy.c
index db363a2d3d66..580e76ae58e6 100644
\-\-\- a/mm/mempolicy\.c
+++ b/mm/mempolicy.c
@@ -2466,7 +2466,7 @@ static void sp\_free(struct sp\_node \*n)

* Policy determination "mimics" alloc\_page\_vma().
* Called from fault path where we know the vma and faulting address.
\*/
-int mpol\_misplaced(struct page \*page, struct vm\_area\_struct \*vma, unsigned long addr)
+int mpol\_misplaced(struct page \*page, struct vm\_area\_struct \*vma, unsigned long addr, int flags)
{
struct mempolicy \*pol;
struct zoneref \*z;
@@ -2477,6 +2477,9 @@ int mpol\_misplaced(struct page \*page, struct vm\_area\_struct \*vma, unsigned long
int polnid = NUMA\_NO\_NODE;
int ret = -1;

* if (test\_and\_clear\_page\_demoted(page))
*
```
  flags |= TNF_DEMOTED;
```

* pol = get\_vma\_policy(vma, addr);
if (!(pol->flags & MPOL\_F\_MOF))
goto out;
@@ -2526,7 +2529,7 @@ int mpol\_misplaced(struct page \*page, struct vm\_area\_struct \*vma, unsigned long
if (pol->flags & MPOL\_F\_MORON) {
polnid = thisnid;

*
```
  if (!should_numa_migrate_memory(current, page, curnid, thiscpu))
```

*
```
  if (!should_numa_migrate_memory(current, page, curnid, thiscpu, flags))
  	goto out;
```

}
## /mm/migrate.c文件
diff --git a/mm/migrate.c b/mm/migrate.c
index fc7f0148fb3f..cda68581e14d 100644
\-\-\- a/mm/migrate\.c
+++ b/mm/migrate.c
@@ -50,6 +50,7 @@
#include \<linux/ptrace.h>
#include \<linux/oom.h>
#include \<linux/memory.h>
+#include <linux/sched/numa\_balancing.h>

#include \<asm/tlbflush.h>

@@ -264,6 +265,15 @@ static bool remove\_migration\_pte(struct page \*page, struct vm\_area\_struct \*vma,
} else
#endif
{
+#ifdef CONFIG\_NUMA\_BALANCING

*
```
  	if (page_is_demoted(page) && vma_migratable(vma)) {
```

*
```
  		bool writable = pte_write(pte);
```

* 
*
```
  		pte = pte_modify(pte, PAGE_NONE);
```

*
```
  		if (writable)
```

*
```
  			pte = pte_mk_savedwrite(pte);
```

*
```
  	}
```

+#endif
set\_pte\_at(vma->vm\_mm, pvmw.address, pvmw.pte, pte);

```
		if (PageAnon(new))
```

@@ -406,6 +416,9 @@ int migrate\_page\_move\_mapping(struct address\_space \*mapping,
int expected\_count = expected\_page\_refs(mapping, page) + extra\_count;
int nr = thp\_nr\_pages(page);

* if (page\_count(page) != expected\_count)
*
```
  count_vm_events(PGMIGRATE_REFCOUNT_FAIL, thp_nr_pages(page));
```

* if (!mapping) {
/\* Anonymous page without mapping \*/
if (page\_count(page) != expected\_count)
@@ -1260,6 +1273,10 @@ static int unmap\_and\_move(new\_page\_t get\_new\_page,
if (!newpage)
return -ENOMEM;
* /\* TODO: check whether Ksm pages can be demoted? \*/
* if (reason == MR\_DEMOTION && !PageKsm(page))
*
```
  set_page_demoted(newpage);
```

* rc = \_\_unmap\_and\_move(page, newpage, force, mode);
if (rc == MIGRATEPAGE\_SUCCESS)
set\_page\_owner\_migrate\_reason(newpage, reason);
@@ -1590,6 +1607,7 @@ int migrate\_pages(struct list\_head \*from, new\_page\_t get\_new\_page,
goto out;
}
nr\_failed++;
*
```
  		count_vm_events(PGMIGRATE_NOMEM_FAIL, thp_nr_pages(page));
  		goto out;
  	case -EAGAIN:
  		if (is_thp) {
```

@@ -2141,8 +2159,10 @@ static int numamigrate\_isolate\_page(pg\_data\_t \*pgdat, struct page \*page)
VM\_BUG\_ON\_PAGE(compound\_order(page) && !PageTransHuge(page), page);

```
/* Avoid migrating to a node that is nearly full */
```

* if (!migrate\_balanced\_pgdat(pgdat, compound\_nr(page)))

* if (!migrate\_balanced\_pgdat(pgdat, compound\_nr(page))) {
*
```
  count_vm_events(PGMIGRATE_DST_NODE_FULL_FAIL, thp_nr_pages(page));
  return 0;
```

* }
if (isolate\_lru\_page(page))
return 0;
@@ -2200,6 +2220,7 @@ int migrate\_misplaced\_page(struct page \*page, struct vm\_area\_struct \*vma,
pg\_data\_t \*pgdat = NODE\_DATA(node);
int isolated;
int nr\_remaining;
* bool is\_file;
LIST\_HEAD(migratepages);
/\*
@@ -2209,18 +2230,15 @@ int migrate\_misplaced\_page(struct page \*page, struct vm\_area\_struct \*vma,
if (is\_shared\_exec\_page(vma, page))
goto out;

* /\*
* 
    * Also do not migrate dirty pages as not all filesystems can move
* 
    * dirty pages in MIGRATE\_ASYNC mode which is a waste of cycles.
* \*/
* if (page\_is\_file\_lru(page) && PageDirty(page))
*
```
  goto out;
```

* isolated = numamigrate\_isolate\_page(pgdat, page);
* if (!isolated)

* if (!isolated) {
*
```
  count_vm_events(PGMIGRATE_NUMA_ISOLATE_FAIL, thp_nr_pages(page));
  goto out;
```

* }
* is\_file = page\_is\_file\_lru(page);
list\_add(&page->lru, &migratepages);
* count\_vm\_numa\_event(PGPROMOTE\_TRIED);
nr\_remaining = migrate\_pages(&migratepages, alloc\_misplaced\_dst\_page,
NULL, node, MIGRATE\_ASYNC,
MR\_NUMA\_MISPLACED, NULL);
@@ -2232,8 +2250,13 @@ int migrate\_misplaced\_page(struct page \*page, struct vm\_area\_struct \*vma,
putback\_lru\_page(page);
}
isolated = 0;

* } else

* } else {
count\_vm\_numa\_event(NUMA\_PAGE\_MIGRATE);
*
```
  if (is_file)
```

*
```
  	count_vm_numa_event(PGPROMOTE_FILE);
```

*
```
  else
```

*
```
  	count_vm_numa_event(PGPROMOTE_ANON);
```

* }
BUG\_ON(!list\_empty(&migratepages));
return isolated;

@@ -2267,13 +2290,16 @@ int migrate\_misplaced\_transhuge\_page(struct mm\_struct \*mm,
new\_page = alloc\_pages\_node(node,
\(GFP\_TRANSHUGE\_LIGHT \| \_\_GFP\_THISNODE\)\,
HPAGE\_PMD\_ORDER);

* if (!new\_page)

* if (!new\_page) {
*
```
  count_vm_events(PGMIGRATE_NOMEM_FAIL, HPAGE_PMD_NR);
  goto out_fail;
```

* }
prep\_transhuge\_page(new\_page);
isolated = numamigrate\_isolate\_page(pgdat, page);
if (!isolated) {
put\_page(new\_page);
*
```
  count_vm_events(PGMIGRATE_NUMA_ISOLATE_FAIL, HPAGE_PMD_NR);
  goto out_fail;
```

}

## /mm/vmscan.c文件
diff --git a/mm/vmscan.c b/mm/vmscan.c
index 62ba2835c74a..47c868d2ecfd 100644
\-\-\- a/mm/vmscan\.c
+++ b/mm/vmscan.c
@@ -1142,6 +1142,7 @@ static unsigned int demote\_page\_list(struct list\_head \*demote\_pages,
int target\_nid = next\_demotion\_node(pgdat->node\_id);
unsigned int nr\_succeeded;
int err;

* bool file\_lru;
if (list\_empty(demote\_pages))
return 0;
@@ -1149,6 +1150,8 @@ static unsigned int demote\_page\_list(struct list\_head \*demote\_pages,
if (target\_nid == NUMA\_NO\_NODE)
return 0;
* file\_lru = page\_is\_file\_lru(lru\_to\_page(demote\_pages));
* /\* Demotion ignores all cpuset and mempolicy settings \*/
err = migrate\_pages(demote\_pages, alloc\_demote\_page, NULL,
target\_nid, MIGRATE\_ASYNC, MR\_DEMOTION,
@@ -1159,6 +1162,11 @@ static unsigned int demote\_page\_list(struct list\_head \*demote\_pages,
else
\_\_count\_vm\_events(PGDEMOTE\_DIRECT, nr\_succeeded);
* if (file\_lru)
*
```
  __count_vm_events(PGDEMOTE_FILE, nr_succeeded);
```

* else
*
```
  __count_vm_events(PGDEMOTE_ANON, nr_succeeded);
```

* return nr\_succeeded;
}

## /mm/vmstat.c
diff --git a/mm/vmstat.c b/mm/vmstat.c
index 90c8c7cbce51..cda2505bb21f 100644
\-\-\- a/mm/vmstat\.c
+++ b/mm/vmstat.c
@@ -1261,6 +1261,8 @@ const char \* const vmstat\_text[] = {
"pgsteal\_direct",
"pgdemote\_kswapd",
"pgdemote\_direct",

* "pgdemote\_file",
* "pgdemote\_anon",
"pgscan\_kswapd",
"pgscan\_direct",
"pgscan\_direct\_throttle",
@@ -1291,10 +1293,21 @@ const char \* const vmstat\_text[] = {
"numa\_hint\_faults",
"numa\_hint\_faults\_local",
"numa\_pages\_migrated",
* "pgpromote\_candidate",
* "pgpromote\_candidate\_demoted",
* "pgpromote\_candidate\_anon",
* "pgpromote\_candidate\_file",
* "pgpromote\_tried",
* "pgpromote\_file",
* "pgpromote\_anon",
#endif
#ifdef CONFIG\_MIGRATION
"pgmigrate\_success",
"pgmigrate\_fail",
* "pgmigrate\_fail\_dst\_node\_full",
* "pgmigrate\_fail\_numa\_isolate",
* "pgmigrate\_fail\_nomem",
* "pgmigrate\_fail\_refcount",
"thp\_migration\_success",
"thp\_migration\_fail",
"thp\_migration\_split",

--
2.30.2

# PATCH 2/5 NUMA balancing for tiered-memory system

With the advent of new memory types and technologies, a server may have
different types of memory, e.g. DRAM, PMEM, CXL-enabled memory, etc. As
different types of memory usually have different level of performance
impact, such a system can be called as a tiered-memory system.

In a tiered-memory system, NUMA memory nodes can be CPU-less nodes. For
such a system, memory with CPU are considered as the toptier node while
memory without CPU are non-toptier nodes.

In default NUMA Balancing, NUMA hint faults are generate on both toptier
and non-toptier nodes. However, in a tiered-memory system, hot memories in
toptier memory nodes may not need to be migrated around. In such cases,
it's unnecessary to scan the pages in the toptier memory nodes. We disable
unnecessary scannings in the toptier nodes for a tiered-memory system.

To support NUMA balancing for a tiered-memory system, the existing sysctl
user-space interface for enabling numa\_balancing is extended in a backward
compatible way, so that users can enable/disable these functionalities
individually. The sysctl is converted from a Boolean value to a bits field.
Current definition for '/proc/sys/kernel/numa\_balancing' is as follow:

* 0x0: NUMA\_BALANCING\_DISABLED
* 0x1: NUMA\_BALANCING\_NORMAL
* 0x2: NUMA\_BALANCING\_TIERED\_MEMORY

If a system has single toptier node online, default NUMA balancing will
automatically be downgraded to the tiered-memory mode to avoid the
unnecessary scanning in the toptier node mentioned above.

## Signed-off-by: Hasan Al Maruf [hasanalmaruf@fb.com](mailto:hasanalmaruf@fb.com)

Documentation/admin\-guide/sysctl/kernel\.rst \| 18 \+\+\+\+\+\+\+\+\+\+\+
include/linux/mempolicy\.h \| 2 \+\+
include/linux/node\.h \| 7 \+\+\+\+
include/linux/sched/sysctl\.h \| 6 \+\+\+\+
kernel/sched/core\.c \| 36 \+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\-\-\-\-
kernel/sched/fair\.c \| 10 \+\+\+\+\+\-
kernel/sched/sched\.h \| 1 \+
kernel/sysctl\.c \| 7 \+\+\-\-
mm/huge\_memory\.c \| 27 \+\+\+\+\+\+\+\+\+\+\-\-\-\-\-\-
mm/mprotect\.c \| 8 \+\+\+\+\-
10 files changed, 101 insertions(+), 21 deletions(-)

## /Documentation/admin-guide/sysctl/kernel.rst文件
@@ -608,6 +608,24 @@ numa\_balancing\_scan\_delay\_ms, numa\_balancing\_scan\_period\_max\_ms,
numa\_balancing\_scan\_size\_mb\`\_, and numa\_balancing\_settle\_count sysctls.

+By default, NUMA hinting faults are generate on both toptier and non-toptier
+nodes. However, in a tiered-memory system, hot memories in toptier memory nodes
+may not need to be migrated around. In such cases, it's unnecessary to scan the
+pages in the toptier memory nodes. For a tiered-memory system, unnecessary scannings
+and hinting faults in the toptier nodes are disabled.
+
+This interface takes bits field as input. Supported values and corresponding modes are
+as follow:
+
+- 0x0: NUMA\_BALANCING\_DISABLED
+- 0x1: NUMA\_BALANCING\_NORMAL
+- 0x2: NUMA\_BALANCING\_TIERED\_MEMORY
+
+If a system has single toptier node online, then default NUMA balancing will
+automatically be downgraded to the tiered-memory mode to avoid the unnecessary scanning
+and hinting faults.
+
+
numa\_balancing\_scan\_period\_min\_ms, numa\_balancing\_scan\_delay\_ms, numa\_balancing\_scan\_period\_max\_ms, numa\_balancing\_scan\_size\_mb

## /include/linux/mempolicy.h文件

@@ -188,6 +188,7 @@ extern int mpol\_misplaced(struct page \*, struct vm\_area\_struct \*, unsigned long,
extern void mpol\_put\_task\_policy(struct task\_struct \*);

extern bool numa\_demotion\_enabled;
+extern bool numa\_promotion\_tiered\_enabled;

#else

@@ -299,5 +300,6 @@ static inline nodemask\_t \*policy\_nodemask\_current(gfp\_t gfp)
}

#define numa\_demotion\_enabled	false
+#define numa\_promotion\_tiered\_enabled	false
#endif /\* CONFIG\_NUMA \*/
#endif

## include/linux/node.h文件

diff --git a/include/linux/node.h b/include/linux/node.h
index 8e5a29897936..9a69b31cae74 100644
\-\-\- a/include/linux/node\.h
+++ b/include/linux/node.h

### register\_hugetlbfs\_with\_node函数

@@ -181,4 +181,11 @@ static inline void register\_hugetlbfs\_with\_node(node\_registration\_func\_t reg,

#define to\_node(device) container\_of(device, struct node, dev)

+static inline bool node\_is\_toptier(int node)
+{

* // ideally, toptier nodes should be the memory with CPU.
* // for now, just assume node0 is the toptier memory
* // return node\_state(node, N\_CPU);
* return (node == 0);
+}

#endif /\* *LINUX\_NODE\_H* \*/
diff --git a/include/linux/sched/sysctl.h b/include/linux/sched/sysctl.h
index 3c31ba88aca5..249e00c42246 100644
\-\-\- a/include/linux/sched/sysctl\.h
+++ b/include/linux/sched/sysctl.h
@@ -39,6 +39,12 @@ enum sched\_tunable\_scaling {
};
extern enum sched\_tunable\_scaling sysctl\_sched\_tunable\_scaling;

+#define NUMA\_BALANCING\_DISABLED			0x0
+#define NUMA\_BALANCING\_NORMAL			0x1
+#define NUMA\_BALANCING\_TIERED\_MEMORY	0x2
+
+extern int sysctl\_numa\_balancing\_mode;
+
extern unsigned int sysctl\_numa\_balancing\_scan\_delay;
extern unsigned int sysctl\_numa\_balancing\_scan\_period\_min;
extern unsigned int sysctl\_numa\_balancing\_scan\_period\_max;
diff --git a/kernel/sched/core.c b/kernel/sched/core.c
index 790c573f7ed4..3d65e601b973 100644
\-\-\- a/kernel/sched/core\.c
+++ b/kernel/sched/core.c
@@ -3596,9 +3596,29 @@ static void \_\_sched\_fork(unsigned long clone\_flags, struct task\_struct \*p)
}

DEFINE\_STATIC\_KEY\_FALSE(sched\_numa\_balancing);
+int sysctl\_numa\_balancing\_mode;
+bool numa\_promotion\_tiered\_enabled;

#ifdef CONFIG\_NUMA\_BALANCING

+/\*

* 
    * If there is only one toptier node available, pages on that
* 
    * node can not be promotrd to anywhere. In that case, downgrade
* 
    * to numa\_promotion\_tiered\_enabled mode
* \*/
+static void check\_numa\_promotion\_mode(void)
+{
* int node, toptier\_node\_count = 0;
* 
* for\_each\_online\_node(node) {
*
```
  if (node_is_toptier(node))
```

*
```
  	++toptier_node_count;
```

* }
* if (toptier\_node\_count == 1) {
*
```
  numa_promotion_tiered_enabled = true;
```

* }
+}
*

void set\_numabalancing\_state(bool enabled)
{
if (enabled)
@@ -3611,20 +3631,22 @@ void set\_numabalancing\_state(bool enabled)
int sysctl\_numa\_balancing(struct ctl\_table \*table, int write,
void \*buffer, size\_t \*lenp, loff\_t \*ppos)
{

* struct ctl\_table t;
int err;
* int state = static\_branch\_likely(&sched\_numa\_balancing);
if (write && !capable(CAP\_SYS\_ADMIN))
return -EPERM;
* t = \*table;
* t.data = &state;
* err = proc\_dointvec\_minmax(&t, write, buffer, lenp, ppos);

* err = proc\_dointvec\_minmax(table, write, buffer, lenp, ppos);
if (err < 0)
return err;

* if (write)
*
```
  set_numabalancing_state(state);
```

* if (write) {
*
```
  if (sysctl_numa_balancing_mode & NUMA_BALANCING_NORMAL)
```

*
```
  	check_numa_promotion_mode();
```

*
```
  else if (sysctl_numa_balancing_mode & NUMA_BALANCING_TIERED_MEMORY)
```

*
```
  	numa_promotion_tiered_enabled = true;
```

* 
*
```
  set_numabalancing_state(*(int *)table->data);
```

* }
return err;
}
#endif
diff --git a/kernel/sched/fair.c b/kernel/sched/fair.c
index 210612c9d1e9..45e39832a2b1 100644
\-\-\- a/kernel/sched/fair\.c
+++ b/kernel/sched/fair.c
@@ -1424,7 +1424,7 @@ bool should\_numa\_migrate\_memory(struct task\_struct \*p, struct page \* page,
count\_vm\_numa\_event(PGPROMOTE\_CANDIDATE);

* if (flags & TNF\_DEMOTED)

* if (numa\_demotion\_enabled && (flags & TNF\_DEMOTED))
count\_vm\_numa\_event(PGPROMOTE\_CANDIDATE\_DEMOTED);
if (page\_is\_file\_lru(page))
@@ -1435,6 +1435,14 @@ bool should\_numa\_migrate\_memory(struct task\_struct \*p, struct page \* page,
this\_cpupid = cpu\_pid\_to\_cpupid(dst\_cpu, current->pid);
last\_cpupid = page\_cpupid\_xchg\_last(page, this\_cpupid);
* /\*
* 
    * The pages in non-toptier memory node should be migrated
* 
    * according to hot/cold instead of accessing CPU node.
* \*/
* if (numa\_promotion\_tiered\_enabled && !node\_is\_toptier(src\_nid))
*
```
  return true;
```

* 
* /\*
    * Allow first faults or private faults to migrate immediately early in
    * the lifetime of a task. The magic number 4 is based on waiting for
    diff --git a/kernel/sched/sched.h b/kernel/sched/sched.h
    index 6057ad67d223..379f3b6f1a3f 100644
    \-\-\- a/kernel/sched/sched\.h
    +++ b/kernel/sched/sched.h
    @@ -51,6 +51,7 @@
    #include \<linux/kthread.h>
    #include \<linux/membarrier.h>
    #include \<linux/migrate.h>
    +#include \<linux/mempolicy.h>
    #include <linux/mm\_inline.h>
    #include <linux/mmu\_context.h>
    #include \<linux/nmi.h>
    diff --git a/kernel/sysctl.c b/kernel/sysctl.c
    index 6b6653529d92..751b52062eb4 100644
    \-\-\- a/kernel/sysctl\.c
    +++ b/kernel/sysctl.c
    @@ -113,6 +113,7 @@ static int sixty = 60;

static int \_\_maybe\_unused neg\_one = -1;
static int \_\_maybe\_unused two = 2;
+static int \_\_maybe\_unused three = 3;
static int \_\_maybe\_unused four = 4;
static unsigned long zero\_ul;
static unsigned long one\_ul = 1;
@@ -1840,12 +1841,12 @@ static struct ctl\_table kern\_table[] = {
},
{
.procname	= "numa\_balancing",

*
```
  .data		= NULL, /* filled in by handler */
```

*
```
  .maxlen		= sizeof(unsigned int),
```

*
```
  .data		= &sysctl_numa_balancing_mode,
```

*
```
  .maxlen		= sizeof(int),
  .mode		= 0644,
  .proc_handler	= sysctl_numa_balancing,
  .extra1		= SYSCTL_ZERO,
```

*
```
  .extra2		= SYSCTL_ONE,
```

*
```
  .extra2		= &three,
```

},
#endif /\* CONFIG\_NUMA\_BALANCING */
#endif /* CONFIG\_SCHED\_DEBUG \*/
diff --git a/mm/huge\_memory.c b/mm/huge\_memory.c
index e9d7b9125c5e..b76a0990c5f1 100644
\-\-\- a/mm/huge\_memory\.c
+++ b/mm/huge\_memory.c
@@ -22,6 +22,7 @@
#include \<linux/freezer.h>
#include <linux/pfn\_t.h>
#include \<linux/mman.h>
+#include \<linux/mempolicy.h>
#include \<linux/memremap.h>
#include \<linux/pagemap.h>
#include \<linux/debugfs.h>
@@ -1849,16 +1850,24 @@ int change\_huge\_pmd(struct vm\_area\_struct \*vma, pmd\_t \*pmd,
}
#endif

* /\*
* 
    * Avoid trapping faults against the zero page. The read-only
* 
    * data is likely to be read-cached on the local CPU and
* 
    * local/remote hits to the zero page are not interesting.
* \*/
* if (prot\_numa && is\_huge\_zero\_pmd(\*pmd))
*
```
  goto unlock;
```

* if (prot\_numa) {
*
```
  struct page *page;
```

*
```
  /*
```

*
```
   * Avoid trapping faults against the zero page. The read-only
```

*
```
   * data is likely to be read-cached on the local CPU and
```

*
```
   * local/remote hits to the zero page are not interesting.
```

*
```
   */
```

*
```
  if (is_huge_zero_pmd(*pmd))
```

*
```
  	goto unlock;
```

* if (prot\_numa && pmd\_protnone(\*pmd))
*
```
  goto unlock;
```

*
```
  if (pmd_protnone(*pmd))
```

*
```
  	goto unlock;
```

* 
*
```
  /* skip scanning toptier node */
```

*
```
  page = pmd_page(*pmd);
```

*
```
  if (numa_promotion_tiered_enabled && node_is_toptier(page_to_nid(page)))
```

*
```
  	goto unlock;
```

* }
/\*
    * In case prot\_numa, we are under mmap\_read\_lock(mm). It's critical
    diff --git a/mm/mprotect.c b/mm/mprotect.c
    index 94188df1ee55..3171f435925b 100644
    \-\-\- a/mm/mprotect\.c
    +++ b/mm/mprotect.c
    @@ -83,6 +83,7 @@ static unsigned long change\_pte\_range(struct vm\_area\_struct \*vma, pmd\_t \*pmd,
    \*/
    if (prot\_numa) {
    struct page \*page;
*
```
  		int nid;

  		/* Avoid TLB flush if possible */
  		if (pte_protnone(oldpte))
```

@@ -109,7 +110,12 @@ static unsigned long change\_pte\_range(struct vm\_area\_struct \*vma, pmd\_t \*pmd,
\* Don't mess with PTEs if page is already on the node
\* a single-threaded process is running on.
\*/

*
```
  		if (target_node == page_to_nid(page))
```

*
```
  		nid = page_to_nid(page);
```

*
```
  		if (target_node == nid)
```

*
```
  			continue;
```

* 
*
```
  		/* skip scanning toptier node */
```

*
```
  		if (numa_promotion_tiered_enabled && node_is_toptier(nid))
  			continue;
  	}
```

# PATCH 3/5

With a tight memory constraint, we need to proactively keep some
free memory in toptier node, such that 1) new allocation which is
mainly for request processing can be directly put in the toptier
node and 2) toptier node is able to accept hot pages promoted from
non-toptier node. To achieve that, we decouple the reclamation and
allocation mechanism, i.e. reclamation gets triggered at a different
watermark -- WMARK\_DEMOTE, while allocation checks for the traditional
WMARK\_HIGH. In this way, toptier nodes can maintain some free space to
accept both new allocation and promotion from non-toptier nodes.
On each toptier memory node, kswapd daemon is woken up to demote memory
when free memory on the node falls below the following fraction
demote\_scale\_factor/10000
The default value of demote\_scale\_factor is 200 , (i.e. 2%) so kswapd will
be woken up when available free memory on a node falls below 2%. The
demote\_scale\_factor can be adjusted higher if we need kswapd to keep more
free memory around by updating the sysctl variable
/proc/sys/vm/demote\_scale\_factor

## 修改文件

Documentation/admin\-guide/sysctl/vm\.rst \| 12 \+\+\+\+\+\+\+\+\+
include/linux/mempolicy\.h \| 5 \+\+\+\+
include/linux/mm\.h \| 4 \+\+\+
include/linux/mmzone\.h \| 5 \+\+\+\+
kernel/sched/fair\.c \| 3 \+\+\+
kernel/sysctl\.c \| 12 \+\+\+\+\+\+\+\+\-
mm/mempolicy\.c \| 23 \+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+
mm/page\_alloc\.c \| 34 \+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\-
mm/vmscan\.c \| 26 \+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+\+
mm/vmstat\.c \| 7 \+\+\+\+\-
10 files changed, 128 insertions(+), 3 deletions(-)

## vm.rst

diff --git a/Documentation/admin-guide/sysctl/vm.rst b/Documentation/admin-guide/sysctl/vm.rst
@@ -74,6 +74,7 @@ Currently, these files are in /proc/sys/vm:

* vfs\_cache\_pressure
* watermark\_boost\_factor
* watermark\_scale\_factor
+- demote\_scale\_factor
* zone\_reclaim\_mode

@@ -961,6 +962,17 @@ that the number of free pages kswapd maintains for latency reasons is
too small for the allocation bursts occurring in the system. This knob
can then be used to tune kswapd aggressiveness accordingly.

+demote\_scale\_factor
+===================
+
+This factor controls when kswapd wakes up to demote pages from toptier
+nodes. It defines the amount of memory left in a toptier node/system
+before kswapd is woken up and how much memory needs to be free from those
+nodes before kswapd goes back to sleep.
+
+The unit is in fractions of 10,000. The default value of 200 means if there
+are less than 2% of free toptier memory in a node/system, we will start to
+demote pages from that node.
zone\_reclaim\_mode

## include/linux/mempolicy.h

diff --git a/include/linux/mempolicy.h b/include/linux/mempolicy.h
@@ -145,6 +145,7 @@ extern void numa\_default\_policy(void);
extern void numa\_policy\_init(void);
extern void mpol\_rebind\_task(struct task\_struct \*tsk, const nodemask\_t \*new);
extern void mpol\_rebind\_mm(struct mm\_struct \*mm, nodemask\_t \*new);
+extern void check\_toptier\_balanced(void);

extern int huge\_node(struct vm\_area\_struct \*vma,
unsigned long addr, gfp\_t gfp\_flags,
@@ -299,6 +300,10 @@ static inline nodemask\_t *policy\_nodemask\_current(gfp\_t gfp)*
return NULL;
}
+static inline void check\_toptier\_balanced(void)
+{
+}
+
#define numa\_demotion\_enabled	false
#define numa\_promotion\_tiered\_enabled	false
#endif / CONFIG\_NUMA \*/

## include/linux/mm.h

@@ -3153,6 +3153,10 @@ static inline bool debug\_guardpage\_enabled(void) { return false; }
static inline bool page\_is\_guard(struct page *page) { return false; }*
#endif / CONFIG\_DEBUG\_PAGEALLOC \*/

+#ifdef CONFIG\_MIGRATION
+extern int demote\_scale\_factor;
+#endif
+
#if MAX\_NUMNODES > 1
void \_\_init setup\_nr\_node\_ids(void);
#else

## include/linux/mmzone.h

@@ -329,12 +329,14 @@ enum zone\_watermarks {
WMARK\_MIN,
WMARK\_LOW,
WMARK\_HIGH,

* WMARK\_DEMOTE,
NR\_WMARK
};

#define min\_wmark\_pages(z) (z->\_watermark[WMARK\_MIN] + z->watermark\_boost)
#define low\_wmark\_pages(z) (z->\_watermark[WMARK\_LOW] + z->watermark\_boost)
#define high\_wmark\_pages(z) (z->\_watermark[WMARK\_HIGH] + z->watermark\_boost)
+#define demote\_wmark\_pages(z) (z->\_watermark[WMARK\_DEMOTE] + z->watermark\_boost)
#define wmark\_pages(z, i) (z->\_watermark[i] + z->watermark\_boost)

struct per\_cpu\_pages {
@@ -884,6 +886,7 @@ bool zone\_watermark\_ok(struct zone \*z, unsigned int order,
unsigned int alloc\_flags);
bool zone\_watermark\_ok\_safe(struct zone \*z, unsigned int order,
unsigned long mark, int highest\_zoneidx);
+bool pgdat\_toptier\_balanced(pg\_data\_t *pgdat, int order, int zone\_idx);*
/

* Memory initialization context, use to differentiate memory added by
* the platform statically or via memory hotplug interface.
@@ -1011,6 +1014,8 @@ int min\_free\_kbytes\_sysctl\_handler(struct ctl\_table \*, int, void \*, size\_t \*,
loff\_t \*);
int watermark\_scale\_factor\_sysctl\_handler(struct ctl\_table \*, int, void \*,
size\_t \*, loff\_t \*);
+int demote\_scale\_factor\_sysctl\_handler(struct ctl\_table \*, int, void \_\_user \*,

*
```
  size_t *, loff_t *);
```

extern int sysctl\_lowmem\_reserve\_ratio[MAX\_NR\_ZONES];
int lowmem\_reserve\_ratio\_sysctl\_handler(struct ctl\_table \*, int, void \*,
size\_t \*, loff\_t \*);

## /kernel/sched/fair.c

@@ -21,6 +21,8 @@

* Copyright (C) 2007 Red Hat, Inc., Peter Zijlstra
\*/
#include "sched.h"
+#include \<trace/events/sched.h>
+#include \<linux/mempolicy.h>

/\*

* Targeted preemption latency for CPU-bound tasks:
@@ -10802,6 +10804,7 @@ void trigger\_load\_balance(struct rq \*rq)
raise\_softirq(SCHED\_SOFTIRQ);
nohz\_balancer\_kick(rq);

* check\_toptier\_balanced();
}

static void rq\_online\_fair(struct rq \*rq)

## /kernel/sysctl.c

@@ -112,6 +112,7 @@ static int sixty = 60;
#endif

static int \_\_maybe\_unused neg\_one = -1;
+static int \_\_maybe\_unused one = 1;
static int \_\_maybe\_unused two = 2;
static int \_\_maybe\_unused three = 3;
static int \_\_maybe\_unused four = 4;
@@ -121,8 +122,8 @@ static unsigned long long\_max = LONG\_MAX;
static int one\_hundred = 100;
static int two\_hundred = 200;
static int one\_thousand = 1000;
-#ifdef CONFIG\_PRINTK
static int ten\_thousand = 10000;
+#ifdef CONFIG\_PRINTK
#endif
#ifdef CONFIG\_PERF\_EVENTS
static int six\_hundred\_forty\_kb = 640 \* 1024;
@@ -3000,6 +3001,15 @@ static struct ctl\_table vm\_table[] = {
.extra1		= SYSCTL\_ONE,
.extra2		= &one\_thousand,
},

* {
*
```
  .procname       = "demote_scale_factor",
```

*
```
  .data           = &demote_scale_factor,
```

*
```
  .maxlen         = sizeof(demote_scale_factor),
```

*
```
  .mode           = 0644,
```

*
```
  .proc_handler   = demote_scale_factor_sysctl_handler,
```

*
```
  .extra1         = &one,
```

*
```
  .extra2         = &ten_thousand,
```

* },
{
.procname	= "percpu\_pagelist\_fraction",
.data		= &percpu\_pagelist\_fraction,

## /mm/mempolicy.c

@@ -1042,6 +1042,29 @@ static long do\_get\_mempolicy(int \*policy, nodemask\_t \*nmask,
return err;
}

+void check\_toptier\_balanced(void)
+{

* int nid;
* int balanced;
* 
* if (!numa\_promotion\_tiered\_enabled)
*
```
  return;
```

* 
* for\_each\_node\_state(nid, N\_MEMORY) {
*
```
  pg_data_t *pgdat = NODE_DATA(nid);
```

* 
*
```
  if (!node_is_toptier(nid))
```

*
```
  	continue;
```

* 
*
```
  balanced = pgdat_toptier_balanced(pgdat, 0, ZONE_MOVABLE);
```

*
```
  if (!balanced) {
```

*
```
  	pgdat->kswapd_order = 0;
```

*
```
  	pgdat->kswapd_highest_zoneidx = ZONE_NORMAL;
```

*
```
  	wakeup_kswapd(pgdat->node_zones + ZONE_NORMAL, 0, 1, ZONE_NORMAL);
```

*
```
  }
```

* }
+}
*

#ifdef CONFIG\_MIGRATION
/\*

* page migration, thp tail pages can be passed.

## /mm/page\_alloc.c

@@ -3599,7 +3599,8 @@ struct page \*rmqueue(struct zone \*preferred\_zone,
if (test\_bit(ZONE\_BOOSTED\_WATERMARK, &zone->flags)) {
clear\_bit(ZONE\_BOOSTED\_WATERMARK, &zone->flags);
wakeup\_kswapd(zone, 0, 0, zone\_idx(zone));

* }

* } else if (!pgdat\_toptier\_balanced(zone->zone\_pgdat, order, zone\_idx(zone)))
*
```
  wakeup_kswapd(zone, 0, 0, zone_idx(zone));
```

VM\_BUG\_ON\_PAGE(page && bad\_range(zone, page), page);
return page;
@@ -8047,6 +8048,22 @@ static void \_\_setup\_per\_zone\_wmarks(void)
zone->\_watermark[WMARK\_LOW] = min\_wmark\_pages(zone) + tmp;
zone->\_watermark[WMARK\_HIGH] = min\_wmark\_pages(zone) + tmp \* 2;
*
```
  if (numa_promotion_tiered_enabled) {
```

*
```
  	tmp = mult_frac(zone_managed_pages(zone), demote_scale_factor, 10000);
```

* 
*
```
  	/*
```

*
```
  	 * Clamp demote watermark between twice high watermark
```

*
```
  	 * and max managed pages.
```

*
```
  	 */
```

*
```
  	if (tmp < 2 * zone->_watermark[WMARK_HIGH])
```

*
```
  		tmp = 2 * zone->_watermark[WMARK_HIGH];
```

*
```
  	if (tmp > zone_managed_pages(zone))
```

*
```
  		tmp = zone_managed_pages(zone);
```

*
```
  	zone->_watermark[WMARK_DEMOTE] = tmp;
```

* 
*
```
  	zone->watermark_boost = 0;
```

*
```
  }
```

*
```
  spin_unlock_irqrestore(&zone->lock, flags);
```

}

@@ -8163,6 +8180,21 @@ int watermark\_scale\_factor\_sysctl\_handler(struct ctl\_table \*table, int write,
return 0;
}

+int demote\_scale\_factor\_sysctl\_handler(struct ctl\_table \*table, int write,

* void \_\_user \*buffer, size\_t \*length, loff\_t \*ppos)
+{
* int rc;
* 
* rc = proc\_dointvec\_minmax(table, write, buffer, length, ppos);
* if (rc)
*
```
  return rc;
```

* 
* if (write)
*
```
  setup_per_zone_wmarks();
```

* 
* return 0;
+}
*

#ifdef CONFIG\_NUMA
static void setup\_min\_unmapped\_ratio(void)
{

## /mm/vmscan.c文件

@@ -41,6 +41,7 @@
#include \<linux/kthread.h>
#include \<linux/freezer.h>
#include \<linux/memcontrol.h>
+#include \<linux/mempolicy.h>
#include \<linux/migrate.h>
#include \<linux/delayacct.h>
#include \<linux/sysctl.h>
@@ -190,6 +191,7 @@ static void set\_task\_reclaim\_state(struct task\_struct \*task,

static LIST\_HEAD(shrinker\_list);
static DECLARE\_RWSEM(shrinker\_rwsem);
+int demote\_scale\_factor = 200;

#ifdef CONFIG\_MEMCG
/\*

### pgdat\_toptier\_balanced函数

@@ -3598,6 +3600,30 @@ static bool pgdat\_balanced(pg\_data\_t \*pgdat, int order, int highest\_zoneidx)
return false;
}

+bool pgdat\_toptier\_balanced(pg\_data\_t \*pgdat, int order, int zone\_idx)
+{

* unsigned long mark;
* struct zone \*zone;
* 
* if \(\!node\_is\_toptier\(pgdat\-\>node\_id\) \|\|
*
```
  !numa_promotion_tiered_enabled ||
```

*
```
  order > 0 || zone_idx < ZONE_NORMAL) {
```

*
```
  return true;
```

* }
* 
* zone = pgdat->node\_zones + ZONE\_NORMAL;
* 
* if (!managed\_zone(zone))
*
```
  return true;
```

* 
* mark = min(demote\_wmark\_pages(zone), zone\_managed\_pages(zone));
* 
* if (zone\_page\_state(zone, NR\_FREE\_PAGES) < mark)
*
```
  return false;
```

* 
* return true;
+}
*

/\* Clear pgdat state for congested, dirty or under writeback. \*/
static void clear\_pgdat\_congested(pg\_data\_t \*pgdat)
{

## /mm/vmstat.c

@@ -28,6 +28,7 @@
#include <linux/mm\_inline.h>
#include <linux/page\_ext.h>
#include <linux/page\_owner.h>
+#include \<linux/migrate.h>

#include "internal.h"

@@ -1649,7 +1650,9 @@ static void zoneinfo\_show\_print(struct seq\_file \*m, pg\_data\_t \*pgdat,
struct zone \*zone)
{
int i;

* seq\_printf(m, "Node %d, zone %8s", pgdat->node\_id, zone->name);

* seq\_printf(m, "Node %d, zone %8s, toptier %d next\_demotion\_node %d",
*
```
  	pgdat->node_id, zone->name, node_is_toptier(pgdat->node_id),
```

*
```
  	next_demotion_node(pgdat->node_id));
```

if (is\_zone\_first\_populated(pgdat, zone)) {
seq\_printf(m, "\n per-node stats");
for (i = 0; i < NR\_VM\_NODE\_STAT\_ITEMS; i++) {
@@ -1666,6 +1669,7 @@ static void zoneinfo\_show\_print(struct seq\_file \*m, pg\_data\_t \*pgdat,
"\n min %lu"
"\n low %lu"
"\n high %lu"
*
```
     "\n        demote   %lu"
     "\n        spanned  %lu"
     "\n        present  %lu"
     "\n        managed  %lu"
```

@@ -1674,6 +1678,7 @@ static void zoneinfo\_show\_print(struct seq\_file \*m, pg\_data\_t \*pgdat,
min\_wmark\_pages(zone),
low\_wmark\_pages(zone),
high\_wmark\_pages(zone),

*
```
     node_is_toptier(pgdat->node_id) ? demote_wmark_pages(zone) : 0,
     zone->spanned_pages,
     zone->present_pages,
     zone_managed_pages(zone),
```

# PATCH 4/5 Reclaim to satisfy WMARK\_DEMOTE on toptier nodes

* demotion：回收上层内存的page，直到到达demotion\_watermark
* 使用kswapd唤起
* 关于THP：开启了THP之后，demotion/promotion可能会延迟几百秒，
    * 原因：顶层页面非常热，kswapd不能回收任何页面
    * 当kswapd失败计数达到最大值，kswapd不会被唤醒，直到成功直接回收发生
    * 一般用例：
    * 分层内存系统：
        * 解决问题，当启用promotion时，kswapd每10s唤醒一次
* When kswapd is wakenup on a toptier node in a tiered-memory NUMA balancing mode, it reclaims pages until the toptier node is balanced and the number of free pages on toptier node satisfies WMARK\_DEMOTE.
* When THP (Transparent Huge Page) is enabled, sometimes demotion/promotion
between the memory nodes may pause for several hundreds of seconds as
the pages in the toptier node may sometimes become so hot, that kswapd
fails to reclaim any page. Finally, the kswapd failure count
(pgdat->kswapd\_failures) reaches its max value and kswapd will not be
waken up until a successful direct reclaiming. For general use case,
this isn't a big problem as the memory users will do direct reclaim
finally and trigger successful direct reclaiming or OOM to fix the
issue. But in memory tiering system, the demotion and promotion will
avoid to create too much memory pressure on the fast memory node, so
direct reclaiming will not be triggered to resolve the issue. To
resolve this, when promotion enabled, kswapd will be waken up every
10 seconds to try to free some pages to recover kswapd failures.

diff --git a/mm/vmscan.c b/mm/vmscan.c

## get\_scan\_count函数

@@ -2386,8 +2386,14 @@ static void get\_scan\_count(struct lruvec \*lruvec, struct scan\_control \*sc,
unsigned long ap, fp;
enum lru\_list lru;

* /\* If we have no swap space, do not bother scanning anon pages. \*/
* if \(\!sc\-\>may\_swap \|\| \!can\_reclaim\_anon\_pages\(memcg\, pgdat\-\>node\_id\, sc\)\) \{

* /\*
* 
    * If we have no swap space, do not bother scanning anon pages.
* 
    * However, anon pages on toptier node can be demoted via reclaim
* 
    * when numa promotion is enabled. Disable the check to prevent
* 
    * demotion for no swap space when numa promotion is enabled.
* \*/
* if (!numa\_promotion\_tiered\_enabled &&
*
```
  (!sc->may_swap || !can_reclaim_anon_pages(memcg, pgdat->node_id, sc))) {
  scan_balance = SCAN_FILE;
  goto out;
```

}

## shrink\_node

@@ -2916,7 +2922,10 @@ static void shrink\_node(pg\_data\_t \*pgdat, struct scan\_control \*sc)
if (!managed\_zone(zone))
continue;

*
```
  	total_high_wmark += high_wmark_pages(zone);
```

*
```
  	if (numa_promotion_tiered_enabled && node_is_toptier(pgdat->node_id))
```

*
```
  		total_high_wmark += demote_wmark_pages(zone);
```

*
```
  	else
```

*
```
  		total_high_wmark += high_wmark_pages(zone);
  }

  /*
```

## pgdat\_balanced

@@ -3574,6 +3583,9 @@ static bool pgdat\_balanced(pg\_data\_t \*pgdat, int order, int highest\_zoneidx)
unsigned long mark = -1;
struct zone \*zone;

* if (numa\_promotion\_tiered\_enabled && node\_is\_toptier(pgdat->node\_id) &&
*
```
  	highest_zoneidx >= ZONE_NORMAL)
```

*
```
  return pgdat_toptier_balanced(pgdat, 0, highest_zoneidx);
```

/\*
    * Check watermarks bottom-up as lower zones are more likely to
    * meet watermarks.

## kswapd\_shrink\_node函数

@@ -3692,7 +3704,10 @@ static bool kswapd\_shrink\_node(pg\_data\_t \*pgdat,
if (!managed\_zone(zone))
continue;

*
```
  sc->nr_to_reclaim += max(high_wmark_pages(zone), SWAP_CLUSTER_MAX);
```

*
```
  if (numa_promotion_tiered_enabled && node_is_toptier(pgdat->node_id))
```

*
```
  	sc->nr_to_reclaim += max(demote_wmark_pages(zone), SWAP_CLUSTER_MAX);
```

*
```
  else
```

*
```
  	sc->nr_to_reclaim += max(high_wmark_pages(zone), SWAP_CLUSTER_MAX);
```

}
/\*

## kswapd\_try\_to\_sleep

@@ -4021,8 +4036,23 @@ static void kswapd\_try\_to\_sleep(pg\_data\_t \*pgdat, int alloc\_order, int reclaim\_o
\*/
set\_pgdat\_percpu\_threshold(pgdat, calculate\_normal\_threshold);

*
```
  if (!kthread_should_stop())
```

*
```
  	schedule();
```

*
```
  if (!kthread_should_stop()) {
```

*
```
  	/*
```

*
```
  	 * In numa promotion modes, try harder to recover from
```

*
```
  	 * kswapd failures, because direct reclaiming may be
```

*
```
  	 * not triggered.
```

*
```
  	 */
```

*
```
  	if (numa_promotion_tiered_enabled &&
```

*
```
  				node_is_toptier(pgdat->node_id) &&
```

*
```
  			pgdat->kswapd_failures >= MAX_RECLAIM_RETRIES) {
```

*
```
  		remaining = schedule_timeout(10 * HZ);
```

*
```
  		if (!remaining) {
```

*
```
  			pgdat->kswapd_highest_zoneidx = ZONE_MOVABLE;
```

*
```
  			pgdat->kswapd_order = 0;
```

*
```
  		}
```

*
```
  	} else
```

*
```
  		schedule();
```

*
```
  }

  set_pgdat_percpu_threshold(pgdat, calculate_pressure_threshold);
```

} else {

--

# PATCH 5/5 active LRU-based promotion to avoid ping-pong

当页面上发生remote hint fault时，默认的NUMA balancing会提升页面，而不检查其状态。
因此，访问非常不频繁的冷页面仍然可以成为提升候选者。一旦被提升到本地节点，这类页面可能很快就会成为降级候选者，如果顶层节点总是处于压力之下的话。
因此，由不经常访问的页面生成的提升流量很容易填满顶层节点的回收空闲空间，并最终为非顶层节点生成更高的降级流量。
这种降级-提升的乒乓效应会在内存节点间产生不必要的流量，并可能对内存密集型应用的性能产生负面影响。

为了解决这个乒乓问题，我们不是立即进行提升，而是通过检查页面在LRU列表中的位置来检查页面的年龄。
如果出错的页面位于非活动LRU中，则我们不会立即考虑该页面为提升候选者，因为它可能是访问不频繁的页面。
我们只考虑位于活动LRUs（无论是匿名还是文件活动LRU）中的出错页面作为提升候选者。
这种方法显著减少了提升流量，并始终在顶层节点上保持满意量的空闲内存，以支持新的分配和来自非顶层节点的提升

## /mm/memory.c

@@ -4202,6 +4202,19 @@ static vm\_fault\_t do\_numa\_page(struct vm\_fault \*vmf)

```
last_cpupid = page_cpupid_last(page);
page_nid = page_to_nid(page);
```

* 
* /\* Only migrate pages that are active on non-toptier node \*/
* if (numa\_promotion\_tiered\_enabled &&
*
```
  !node_is_toptier(page_nid) &&
```

*
```
  !PageActive(page)) {
```

*
```
  count_vm_numa_event(NUMA_HINT_FAULTS);
```

*
```
  if (page_nid == numa_node_id())
```

*
```
  	count_vm_numa_event(NUMA_HINT_FAULTS_LOCAL);
```

*
```
  mark_page_accessed(page);
```

*
```
  pte_unmap_unlock(vmf->pte, vmf->ptl);
```

*
```
  goto out;
```

* }
* target\_nid = numa\_migrate\_prep(page, vma, vmf->address, page\_nid,
&flags);
pte\_unmap\_unlock(vmf->pte, vmf->ptl);