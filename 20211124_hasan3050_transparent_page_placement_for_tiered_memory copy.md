
# [PATCH 1/5] Promotion and demotion related statistics

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

To track the demoted pages, PG_demoted bit is introduced for pages that
get demoted. Upon demotion, PG_demoted bit is set in thepage flag. upon
promotion, the bit gets reset for that page.

promotion related statistics:
pgpromote_candidate - candidates that get selected for promotion
pgpromote_candidate_demoted - promotion candidate that got demoted earlier
pgpromote_candidate_anon - promotion candidate that are anon
pgpromote_candidate_file - promotion candidate that are file
pgpromote_tried - pages that had a try to migrate via NUMA Balancing
pgpromote_file- successfully promoted file pages
pgpromote_anon - successfully promoted anon pages

promotion failure related statistics:
pgmigrate_fail_dst_node_full - failed as the target node is full
pgmigrate_fail_numa_isolate - failed in isolating numa page
pgmigrate_fail_nomem - failed as no memory left in the system
pgmigrate_fail_refcount - failed as ref count mismatched

demotion related statistics:
pgdemote_file - successfully demoted file pages
pgdemote_anon - successfully demoted anon pages

## 修改文件
 include/linux/mempolicy.h            |  4 +-
 include/linux/page-flags.h           |  9 ++++
 include/linux/page_ext.h             |  3 ++
 include/linux/sched/numa_balancing.h | 63 +++++++++++++++++++++++++++-
 include/linux/vm_event_item.h        | 13 ++++++
 include/trace/events/mmflags.h       | 10 ++++-
 kernel/sched/fair.c                  | 12 +++++-
 kernel/sched/sched.h                 |  1 +
 mm/huge_memory.c                     |  2 +-
 mm/memory.c                          |  2 +-
 mm/mempolicy.c                       |  7 +++-
 mm/migrate.c                         | 48 ++++++++++++++++-----
 mm/vmscan.c                          |  8 ++++
 mm/vmstat.c                          | 13 ++++++
 14 files changed, 174 insertions(+), 21 deletions(-)

## /include/linux/mempolicy.h
index 78a736e76d5c..c7637cfa1be2 100644
--- a/include/linux/mempolicy.h
+++ b/include/linux/mempolicy.h
@@ -184,7 +184,7 @@ extern void mpol_to_str(char *buffer, int maxlen, struct mempolicy *pol);
 /* Check if a vma is migratable */
 extern bool vma_migratable(struct vm_area_struct *vma);
 
-extern int mpol_misplaced(struct page *, struct vm_area_struct *, unsigned long);
+extern int mpol_misplaced(struct page *, struct vm_area_struct *, unsigned long, int);
 extern void mpol_put_task_policy(struct task_struct *);
 
 extern bool numa_demotion_enabled;
@@ -284,7 +284,7 @@ static inline int mpol_parse_str(char *str, struct mempolicy **mpol)
 #endif
 
 static inline int mpol_misplaced(struct page *page, struct vm_area_struct *vma,
-				 unsigned long address)
+				 unsigned long address, int flags)
 {
 	return -1; /* no node preference */
 }
## /include/linux/page-flags.h
index 04a34c08e0a6..8babc550d178 100644
--- a/include/linux/page-flags.h
+++ b/include/linux/page-flags.h
@@ -137,6 +137,9 @@ enum pageflags {
 #endif
 #ifdef CONFIG_64BIT
 	PG_arch_2,
+#ifdef CONFIG_NUMA_BALANCING
+	PG_demoted,
+#endif
 #endif
 	__NR_PAGEFLAGS,
 
@@ -443,6 +446,12 @@ TESTCLEARFLAG(Young, young, PF_ANY)
 PAGEFLAG(Idle, idle, PF_ANY)
 #endif
 
+#if defined(CONFIG_NUMA_BALANCING) && defined(CONFIG_64BIT)
+TESTPAGEFLAG(Demoted, demoted, PF_NO_TAIL)
+SETPAGEFLAG(Demoted, demoted, PF_NO_TAIL)
+TESTCLEARFLAG(Demoted, demoted, PF_NO_TAIL)
+#endif
+
 /*
  * PageReported() is used to track reported free pages within the Buddy
  * allocator. We can use the non-atomic version of the test and set
diff --git a/include/linux/page_ext.h b/include/linux/page_ext.h
index aff81ba31bd8..1a1e632031d3 100644
--- a/include/linux/page_ext.h
+++ b/include/linux/page_ext.h
@@ -23,6 +23,9 @@ enum page_ext_flags {
 	PAGE_EXT_YOUNG,
 	PAGE_EXT_IDLE,
 #endif
+#if defined(CONFIG_NUMA_BALANCING) && !defined(CONFIG_64BIT)
+	PAGE_EXT_DEMOTED,
+#endif
 };
 
 /*

## /include/linux/sched/numa_balancing.h
index 3988762efe15..c13ba820c07d 100644
--- a/include/linux/sched/numa_balancing.h
+++ b/include/linux/sched/numa_balancing.h
@@ -8,12 +8,14 @@
  */
 
 #include <linux/sched.h>
+#include <linux/page-flags.h>
 
 #define TNF_MIGRATED	0x01
 #define TNF_NO_GROUP	0x02
 #define TNF_SHARED	0x04
 #define TNF_FAULT_LOCAL	0x08
 #define TNF_MIGRATE_FAIL 0x10
+#define TNF_DEMOTED	0x40
 
 #ifdef CONFIG_NUMA_BALANCING
 extern void task_numa_fault(int last_node, int node, int pages, int flags);
@@ -21,7 +23,53 @@ extern pid_t task_numa_group_id(struct task_struct *p);
 extern void set_numabalancing_state(bool enabled);
 extern void task_numa_free(struct task_struct *p, bool final);
 extern bool should_numa_migrate_memory(struct task_struct *p, struct page *page,
-					int src_nid, int dst_cpu);
+					int src_nid, int dst_cpu, int flags);
+#ifdef CONFIG_64BIT
+static inline bool page_is_demoted(struct page *page)
+{
+	return PageDemoted(page);
+}
+
+static inline void set_page_demoted(struct page *page)
+{
+	SetPageDemoted(page);
+}
+
+static inline bool test_and_clear_page_demoted(struct page *page)
+{
+	return TestClearPageDemoted(page);
+}
+#else /* !CONFIG_64BIT */
+static inline bool page_is_demoted(struct page *page)
+{
+	struct page_ext *page_ext = lookup_page_ext(page);
+
+	if (unlikely(!page_ext))
+		return false;
+
+	return test_bit(PAGE_EXT_DEMOTED, &page_ext->flags);
+}
+
+static inline void set_page_demoted(struct page *page)
+{
+	struct page_ext *page_ext = lookup_page_ext(page);
+
+	if (unlikely(!page_ext))
+		return false;
+
+	return set_bit(PAGE_EXT_DEMOTED, &page_ext->flags);
+}
+
+static inline bool test_and_clear_page_demoted(struct page *page)
+{
+	struct page_ext *page_ext = lookup_page_ext(page);
+
+	if (unlikely(!page_ext))
+		return false;
+
+	return test_and_clear_bit(PAGE_EXT_DEMOTED, &page_ext->flags);
+}
+#endif /* !CONFIG_64BIT */
 #else
 static inline void task_numa_fault(int last_node, int node, int pages,
 				   int flags)
@@ -38,10 +86,21 @@ static inline void task_numa_free(struct task_struct *p, bool final)
 {
 }
 static inline bool should_numa_migrate_memory(struct task_struct *p,
-				struct page *page, int src_nid, int dst_cpu)
+				struct page *page, int src_nid, int dst_cpu, int flags)
 {
 	return true;
 }
+static inline bool page_is_demoted(struct page *page)
+{
+	return false;
+}
+static inline void set_page_demoted(struct page *page)
+{
+}
+static inline bool test_and_clear_page_demoted(struct page *page)
+{
+	return false;
+}
 #endif
 
 #endif /* _LINUX_SCHED_NUMA_BALANCING_H */
diff --git a/include/linux/vm_event_item.h b/include/linux/vm_event_item.h
index b136ed6224a2..9cb43a2998cb 100644
--- a/include/linux/vm_event_item.h
+++ b/include/linux/vm_event_item.h
@@ -35,6 +35,8 @@ enum vm_event_item { PGPGIN, PGPGOUT, PSWPIN, PSWPOUT,
 		PGSTEAL_DIRECT,
 		PGDEMOTE_KSWAPD,
 		PGDEMOTE_DIRECT,
+		PGDEMOTE_FILE,
+		PGDEMOTE_ANON,
 		PGSCAN_KSWAPD,
 		PGSCAN_DIRECT,
 		PGSCAN_DIRECT_THROTTLE,
@@ -56,9 +58,20 @@ enum vm_event_item { PGPGIN, PGPGOUT, PSWPIN, PSWPOUT,
 		NUMA_HINT_FAULTS,
 		NUMA_HINT_FAULTS_LOCAL,
 		NUMA_PAGE_MIGRATE,
+		PGPROMOTE_CANDIDATE,		/* candidates get selected for promotion */
+		PGPROMOTE_CANDIDATE_DEMOTED,/* promotion candidate that got demoted earlier */
+		PGPROMOTE_CANDIDATE_ANON,	/* promotion candidate that are anon */
+		PGPROMOTE_CANDIDATE_FILE,	/* promotion candidate that are file */
+		PGPROMOTE_TRIED,			/* tried to migrate via NUMA balancing */
+		PGPROMOTE_FILE,				/* successfully promoted file pages  */
+		PGPROMOTE_ANON,				/* successfully promoted anon pages  */
 #endif
 #ifdef CONFIG_MIGRATION
 		PGMIGRATE_SUCCESS, PGMIGRATE_FAIL,
+		PGMIGRATE_DST_NODE_FULL_FAIL,	/* failed as the target node is full */
+		PGMIGRATE_NUMA_ISOLATE_FAIL,	/* failed in isolating numa page */
+		PGMIGRATE_NOMEM_FAIL,			/* failed as no memory left */
+		PGMIGRATE_REFCOUNT_FAIL,		/* failed in ref count */
 		THP_MIGRATION_SUCCESS,
 		THP_MIGRATION_FAIL,
 		THP_MIGRATION_SPLIT,
diff --git a/include/trace/events/mmflags.h b/include/trace/events/mmflags.h
index 67018d367b9f..7ba2c2702ef7 100644
--- a/include/trace/events/mmflags.h
+++ b/include/trace/events/mmflags.h
@@ -85,6 +85,13 @@
 #define IF_HAVE_PG_ARCH_2(flag,string)
 #endif
 
+#if defined(CONFIG_NUMA_BALANCING) && defined(CONFIG_64BIT)
+#define IF_HAVE_PG_DEMOTED(flag, string) ,{1UL << flag, string}
+#else
+#define IF_HAVE_PG_DEMOTED(flag, string)
+#endif
+
+
 #define __def_pageflag_names						\
 	{1UL << PG_locked,		"locked"	},		\
 	{1UL << PG_waiters,		"waiters"	},		\
@@ -112,7 +119,8 @@ IF_HAVE_PG_UNCACHED(PG_uncached,	"uncached"	)		\
 IF_HAVE_PG_HWPOISON(PG_hwpoison,	"hwpoison"	)		\
 IF_HAVE_PG_IDLE(PG_young,		"young"		)		\
 IF_HAVE_PG_IDLE(PG_idle,		"idle"		)		\
-IF_HAVE_PG_ARCH_2(PG_arch_2,		"arch_2"	)
+IF_HAVE_PG_ARCH_2(PG_arch_2,		"arch_2")	\
+IF_HAVE_PG_DEMOTED(PG_demoted,		"demoted")
 
 #define show_page_flags(flags)						\
 	(flags) ? __print_flags(flags, "|",				\
diff --git a/kernel/sched/fair.c b/kernel/sched/fair.c
index 572f312cc803..210612c9d1e9 100644
--- a/kernel/sched/fair.c
+++ b/kernel/sched/fair.c
@@ -1416,12 +1416,22 @@ static inline unsigned long group_weight(struct task_struct *p, int nid,
 }
 
 bool should_numa_migrate_memory(struct task_struct *p, struct page * page,
-				int src_nid, int dst_cpu)
+				int src_nid, int dst_cpu, int flags)
 {
 	struct numa_group *ng = deref_curr_numa_group(p);
 	int dst_nid = cpu_to_node(dst_cpu);
 	int last_cpupid, this_cpupid;
 
+	count_vm_numa_event(PGPROMOTE_CANDIDATE);
+
+	if (flags & TNF_DEMOTED)
+		count_vm_numa_event(PGPROMOTE_CANDIDATE_DEMOTED);
+
+	if (page_is_file_lru(page))
+		count_vm_numa_event(PGPROMOTE_CANDIDATE_FILE);
+	else
+		count_vm_numa_event(PGPROMOTE_CANDIDATE_ANON);
+
 	this_cpupid = cpu_pid_to_cpupid(dst_cpu, current->pid);
 	last_cpupid = page_cpupid_xchg_last(page, this_cpupid);
 
diff --git a/kernel/sched/sched.h b/kernel/sched/sched.h
index eee49ce2d596..6057ad67d223 100644
--- a/kernel/sched/sched.h
+++ b/kernel/sched/sched.h
@@ -51,6 +51,7 @@
 #include <linux/kthread.h>
 #include <linux/membarrier.h>
 #include <linux/migrate.h>
+#include <linux/mm_inline.h>
 #include <linux/mmu_context.h>
 #include <linux/nmi.h>
 #include <linux/proc_fs.h>
diff --git a/mm/huge_memory.c b/mm/huge_memory.c
index bc642923e0c9..e9d7b9125c5e 100644
--- a/mm/huge_memory.c
+++ b/mm/huge_memory.c
@@ -1475,7 +1475,7 @@ vm_fault_t do_huge_pmd_numa_page(struct vm_fault *vmf, pmd_t pmd)
 	 * page_table_lock if at all possible
 	 */
 	page_locked = trylock_page(page);
-	target_nid = mpol_misplaced(page, vma, haddr);
+	target_nid = mpol_misplaced(page, vma, haddr, flags);
 	if (target_nid == NUMA_NO_NODE) {
 		/* If the page was locked, there are no parallel migrations */
 		if (page_locked)
diff --git a/mm/memory.c b/mm/memory.c
index c8083f571c89..314fe3b2f462 100644
--- a/mm/memory.c
+++ b/mm/memory.c
@@ -4131,7 +4131,7 @@ static int numa_migrate_prep(struct page *page, struct vm_area_struct *vma,
 		*flags |= TNF_FAULT_LOCAL;
 	}
 
-	return mpol_misplaced(page, vma, addr);
+	return mpol_misplaced(page, vma, addr, *flags);
 }
 
 static vm_fault_t do_numa_page(struct vm_fault *vmf)
diff --git a/mm/mempolicy.c b/mm/mempolicy.c
index db363a2d3d66..580e76ae58e6 100644
--- a/mm/mempolicy.c
+++ b/mm/mempolicy.c
@@ -2466,7 +2466,7 @@ static void sp_free(struct sp_node *n)
  * Policy determination "mimics" alloc_page_vma().
  * Called from fault path where we know the vma and faulting address.
  */
-int mpol_misplaced(struct page *page, struct vm_area_struct *vma, unsigned long addr)
+int mpol_misplaced(struct page *page, struct vm_area_struct *vma, unsigned long addr, int flags)
 {
 	struct mempolicy *pol;
 	struct zoneref *z;
@@ -2477,6 +2477,9 @@ int mpol_misplaced(struct page *page, struct vm_area_struct *vma, unsigned long
 	int polnid = NUMA_NO_NODE;
 	int ret = -1;
 
+	if (test_and_clear_page_demoted(page))
+		flags |= TNF_DEMOTED;
+
 	pol = get_vma_policy(vma, addr);
 	if (!(pol->flags & MPOL_F_MOF))
 		goto out;
@@ -2526,7 +2529,7 @@ int mpol_misplaced(struct page *page, struct vm_area_struct *vma, unsigned long
 	if (pol->flags & MPOL_F_MORON) {
 		polnid = thisnid;
 
-		if (!should_numa_migrate_memory(current, page, curnid, thiscpu))
+		if (!should_numa_migrate_memory(current, page, curnid, thiscpu, flags))
 			goto out;
 	}
 
diff --git a/mm/migrate.c b/mm/migrate.c
index fc7f0148fb3f..cda68581e14d 100644
--- a/mm/migrate.c
+++ b/mm/migrate.c
@@ -50,6 +50,7 @@
 #include <linux/ptrace.h>
 #include <linux/oom.h>
 #include <linux/memory.h>
+#include <linux/sched/numa_balancing.h>
 
 #include <asm/tlbflush.h>
 
@@ -264,6 +265,15 @@ static bool remove_migration_pte(struct page *page, struct vm_area_struct *vma,
 		} else
 #endif
 		{
+#ifdef CONFIG_NUMA_BALANCING
+			if (page_is_demoted(page) && vma_migratable(vma)) {
+				bool writable = pte_write(pte);
+
+				pte = pte_modify(pte, PAGE_NONE);
+				if (writable)
+					pte = pte_mk_savedwrite(pte);
+			}
+#endif
 			set_pte_at(vma->vm_mm, pvmw.address, pvmw.pte, pte);
 
 			if (PageAnon(new))
@@ -406,6 +416,9 @@ int migrate_page_move_mapping(struct address_space *mapping,
 	int expected_count = expected_page_refs(mapping, page) + extra_count;
 	int nr = thp_nr_pages(page);
 
+	if (page_count(page) != expected_count)
+		count_vm_events(PGMIGRATE_REFCOUNT_FAIL, thp_nr_pages(page));
+
 	if (!mapping) {
 		/* Anonymous page without mapping */
 		if (page_count(page) != expected_count)
@@ -1260,6 +1273,10 @@ static int unmap_and_move(new_page_t get_new_page,
 	if (!newpage)
 		return -ENOMEM;
 
+	/* TODO: check whether Ksm pages can be demoted? */
+	if (reason == MR_DEMOTION && !PageKsm(page))
+		set_page_demoted(newpage);
+
 	rc = __unmap_and_move(page, newpage, force, mode);
 	if (rc == MIGRATEPAGE_SUCCESS)
 		set_page_owner_migrate_reason(newpage, reason);
@@ -1590,6 +1607,7 @@ int migrate_pages(struct list_head *from, new_page_t get_new_page,
 					goto out;
 				}
 				nr_failed++;
+				count_vm_events(PGMIGRATE_NOMEM_FAIL, thp_nr_pages(page));
 				goto out;
 			case -EAGAIN:
 				if (is_thp) {
@@ -2141,8 +2159,10 @@ static int numamigrate_isolate_page(pg_data_t *pgdat, struct page *page)
 	VM_BUG_ON_PAGE(compound_order(page) && !PageTransHuge(page), page);
 
 	/* Avoid migrating to a node that is nearly full */
-	if (!migrate_balanced_pgdat(pgdat, compound_nr(page)))
+	if (!migrate_balanced_pgdat(pgdat, compound_nr(page))) {
+		count_vm_events(PGMIGRATE_DST_NODE_FULL_FAIL, thp_nr_pages(page));
 		return 0;
+	}
 
 	if (isolate_lru_page(page))
 		return 0;
@@ -2200,6 +2220,7 @@ int migrate_misplaced_page(struct page *page, struct vm_area_struct *vma,
 	pg_data_t *pgdat = NODE_DATA(node);
 	int isolated;
 	int nr_remaining;
+	bool is_file;
 	LIST_HEAD(migratepages);
 
 	/*
@@ -2209,18 +2230,15 @@ int migrate_misplaced_page(struct page *page, struct vm_area_struct *vma,
 	if (is_shared_exec_page(vma, page))
 		goto out;
 
-	/*
-	 * Also do not migrate dirty pages as not all filesystems can move
-	 * dirty pages in MIGRATE_ASYNC mode which is a waste of cycles.
-	 */
-	if (page_is_file_lru(page) && PageDirty(page))
-		goto out;
-
 	isolated = numamigrate_isolate_page(pgdat, page);
-	if (!isolated)
+	if (!isolated) {
+		count_vm_events(PGMIGRATE_NUMA_ISOLATE_FAIL, thp_nr_pages(page));
 		goto out;
+	}
 
+	is_file = page_is_file_lru(page);
 	list_add(&page->lru, &migratepages);
+	count_vm_numa_event(PGPROMOTE_TRIED);
 	nr_remaining = migrate_pages(&migratepages, alloc_misplaced_dst_page,
 				     NULL, node, MIGRATE_ASYNC,
 				     MR_NUMA_MISPLACED, NULL);
@@ -2232,8 +2250,13 @@ int migrate_misplaced_page(struct page *page, struct vm_area_struct *vma,
 			putback_lru_page(page);
 		}
 		isolated = 0;
-	} else
+	} else {
 		count_vm_numa_event(NUMA_PAGE_MIGRATE);
+		if (is_file)
+			count_vm_numa_event(PGPROMOTE_FILE);
+		else
+			count_vm_numa_event(PGPROMOTE_ANON);
+	}
 	BUG_ON(!list_empty(&migratepages));
 	return isolated;
 
@@ -2267,13 +2290,16 @@ int migrate_misplaced_transhuge_page(struct mm_struct *mm,
 	new_page = alloc_pages_node(node,
 		(GFP_TRANSHUGE_LIGHT | __GFP_THISNODE),
 		HPAGE_PMD_ORDER);
-	if (!new_page)
+	if (!new_page) {
+		count_vm_events(PGMIGRATE_NOMEM_FAIL, HPAGE_PMD_NR);
 		goto out_fail;
+	}
 	prep_transhuge_page(new_page);
 
 	isolated = numamigrate_isolate_page(pgdat, page);
 	if (!isolated) {
 		put_page(new_page);
+		count_vm_events(PGMIGRATE_NUMA_ISOLATE_FAIL, HPAGE_PMD_NR);
 		goto out_fail;
 	}
 
## /mm/vmscan.c
@@ -1142,6 +1142,7 @@ static unsigned int demote_page_list(struct list_head *demote_pages,
 	int target_nid = next_demotion_node(pgdat->node_id);
 	unsigned int nr_succeeded;
 	int err;
+	bool file_lru;
 
 	if (list_empty(demote_pages))
 		return 0;
@@ -1149,6 +1150,8 @@ static unsigned int demote_page_list(struct list_head *demote_pages,
 	if (target_nid == NUMA_NO_NODE)
 		return 0;
 
+	file_lru = page_is_file_lru(lru_to_page(demote_pages));
+
 	/* Demotion ignores all cpuset and mempolicy settings */
 	err = migrate_pages(demote_pages, alloc_demote_page, NULL,
 			    target_nid, MIGRATE_ASYNC, MR_DEMOTION,
@@ -1159,6 +1162,11 @@ static unsigned int demote_page_list(struct list_head *demote_pages,
 	else
 		__count_vm_events(PGDEMOTE_DIRECT, nr_succeeded);
 
+	if (file_lru)
+		__count_vm_events(PGDEMOTE_FILE, nr_succeeded);
+	else
+		__count_vm_events(PGDEMOTE_ANON, nr_succeeded);
+
 	return nr_succeeded;
 }

## /mm/vmstat.c
index 90c8c7cbce51..cda2505bb21f 100644
--- a/mm/vmstat.c
+++ b/mm/vmstat.c
@@ -1261,6 +1261,8 @@ const char * const vmstat_text[] = {
 	"pgsteal_direct",
 	"pgdemote_kswapd",
 	"pgdemote_direct",
+	"pgdemote_file",
+	"pgdemote_anon",
 	"pgscan_kswapd",
 	"pgscan_direct",
 	"pgscan_direct_throttle",
@@ -1291,10 +1293,21 @@ const char * const vmstat_text[] = {
 	"numa_hint_faults",
 	"numa_hint_faults_local",
 	"numa_pages_migrated",
+	"pgpromote_candidate",
+	"pgpromote_candidate_demoted",
+	"pgpromote_candidate_anon",
+	"pgpromote_candidate_file",
+	"pgpromote_tried",
+	"pgpromote_file",
+	"pgpromote_anon",
 #endif
 #ifdef CONFIG_MIGRATION
 	"pgmigrate_success",
 	"pgmigrate_fail",
+	"pgmigrate_fail_dst_node_full",
+	"pgmigrate_fail_numa_isolate",
+	"pgmigrate_fail_nomem",
+	"pgmigrate_fail_refcount",
 	"thp_migration_success",
 	"thp_migration_fail",
 	"thp_migration_split",



# [PATCH 2/5] NUMA balancing for tiered-memory system


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
user-space interface for enabling numa_balancing is extended in a backward
compatible way, so that users can enable/disable these functionalities
individually. The sysctl is converted from a Boolean value to a bits field.
Current definition for '/proc/sys/kernel/numa_balancing' is as follow:

- 0x0: NUMA_BALANCING_DISABLED
- 0x1: NUMA_BALANCING_NORMAL
- 0x2: NUMA_BALANCING_TIERED_MEMORY

If a system has single toptier node online, default NUMA balancing will
automatically be downgraded to the tiered-memory mode to avoid the
unnecessary scanning in the toptier node mentioned above.

## 修改文件
 Documentation/admin-guide/sysctl/kernel.rst | 18 +++++++++++
 include/linux/mempolicy.h                   |  2 ++
 include/linux/node.h                        |  7 ++++
 include/linux/sched/sysctl.h                |  6 ++++
 kernel/sched/core.c                         | 36 +++++++++++++++++----
 kernel/sched/fair.c                         | 10 +++++-
 kernel/sched/sched.h                        |  1 +
 kernel/sysctl.c                             |  7 ++--
 mm/huge_memory.c                            | 27 ++++++++++------
 mm/mprotect.c                               |  8 ++++-
 10 files changed, 101 insertions(+), 21 deletions(-)

diff --git a/Documentation/admin-guide/sysctl/kernel.rst b/Documentation/admin-guide/sysctl/kernel.rst
index 24ab20d7a50a..1abab69dd5b6 100644
--- a/Documentation/admin-guide/sysctl/kernel.rst
+++ b/Documentation/admin-guide/sysctl/kernel.rst
@@ -608,6 +608,24 @@ numa_balancing_scan_delay_ms, numa_balancing_scan_period_max_ms,
 numa_balancing_scan_size_mb`_, and numa_balancing_settle_count sysctls.
 
 
+By default, NUMA hinting faults are generate on both toptier and non-toptier
+nodes. However, in a tiered-memory system, hot memories in toptier memory nodes
+may not need to be migrated around. In such cases, it's unnecessary to scan the
+pages in the toptier memory nodes. For a tiered-memory system, unnecessary scannings
+and hinting faults in the toptier nodes are disabled.
+
+This interface takes bits field as input. Supported values and corresponding modes are
+as follow:
+
+- 0x0: NUMA_BALANCING_DISABLED
+- 0x1: NUMA_BALANCING_NORMAL
+- 0x2: NUMA_BALANCING_TIERED_MEMORY
+
+If a system has single toptier node online, then default NUMA balancing will
+automatically be downgraded to the tiered-memory mode to avoid the unnecessary scanning
+and hinting faults.
+
+
 numa_balancing_scan_period_min_ms, numa_balancing_scan_delay_ms, numa_balancing_scan_period_max_ms, numa_balancing_scan_size_mb

 
diff --git a/include/linux/mempolicy.h b/include/linux/mempolicy.h
index c7637cfa1be2..ab57b6a82e0a 100644
--- a/include/linux/mempolicy.h
+++ b/include/linux/mempolicy.h
@@ -188,6 +188,7 @@ extern int mpol_misplaced(struct page *, struct vm_area_struct *, unsigned long,
 extern void mpol_put_task_policy(struct task_struct *);
 
 extern bool numa_demotion_enabled;
+extern bool numa_promotion_tiered_enabled;
 
 #else
 
@@ -299,5 +300,6 @@ static inline nodemask_t *policy_nodemask_current(gfp_t gfp)
 }
 
 #define numa_demotion_enabled	false
+#define numa_promotion_tiered_enabled	false
 #endif /* CONFIG_NUMA */
 #endif
diff --git a/include/linux/node.h b/include/linux/node.h
index 8e5a29897936..9a69b31cae74 100644
--- a/include/linux/node.h
+++ b/include/linux/node.h
@@ -181,4 +181,11 @@ static inline void register_hugetlbfs_with_node(node_registration_func_t reg,
 
 #define to_node(device) container_of(device, struct node, dev)
 
+static inline bool node_is_toptier(int node)
+{
+	// ideally, toptier nodes should be the memory with CPU.
+	// for now, just assume node0 is the toptier memory
+	// return node_state(node, N_CPU);
+	return (node == 0);
+}
 #endif /* _LINUX_NODE_H_ */
diff --git a/include/linux/sched/sysctl.h b/include/linux/sched/sysctl.h
index 3c31ba88aca5..249e00c42246 100644
--- a/include/linux/sched/sysctl.h
+++ b/include/linux/sched/sysctl.h
@@ -39,6 +39,12 @@ enum sched_tunable_scaling {
 };
 extern enum sched_tunable_scaling sysctl_sched_tunable_scaling;
 
+#define NUMA_BALANCING_DISABLED			0x0
+#define NUMA_BALANCING_NORMAL			0x1
+#define NUMA_BALANCING_TIERED_MEMORY	0x2
+
+extern int sysctl_numa_balancing_mode;
+
 extern unsigned int sysctl_numa_balancing_scan_delay;
 extern unsigned int sysctl_numa_balancing_scan_period_min;
 extern unsigned int sysctl_numa_balancing_scan_period_max;
diff --git a/kernel/sched/core.c b/kernel/sched/core.c
index 790c573f7ed4..3d65e601b973 100644
--- a/kernel/sched/core.c
+++ b/kernel/sched/core.c
@@ -3596,9 +3596,29 @@ static void __sched_fork(unsigned long clone_flags, struct task_struct *p)
 }
 
 DEFINE_STATIC_KEY_FALSE(sched_numa_balancing);
+int sysctl_numa_balancing_mode;
+bool numa_promotion_tiered_enabled;
 
 #ifdef CONFIG_NUMA_BALANCING
 
+/*
+ * If there is only one toptier node available, pages on that
+ * node can not be promotrd to anywhere. In that case, downgrade
+ * to numa_promotion_tiered_enabled mode
+ */
+static void check_numa_promotion_mode(void)
+{
+	int node, toptier_node_count = 0;
+
+	for_each_online_node(node) {
+		if (node_is_toptier(node))
+			++toptier_node_count;
+	}
+	if (toptier_node_count == 1) {
+		numa_promotion_tiered_enabled = true;
+	}
+}
+
 void set_numabalancing_state(bool enabled)
 {
 	if (enabled)
@@ -3611,20 +3631,22 @@ void set_numabalancing_state(bool enabled)
 int sysctl_numa_balancing(struct ctl_table *table, int write,
 			  void *buffer, size_t *lenp, loff_t *ppos)
 {
-	struct ctl_table t;
 	int err;
-	int state = static_branch_likely(&sched_numa_balancing);
 
 	if (write && !capable(CAP_SYS_ADMIN))
 		return -EPERM;
 
-	t = *table;
-	t.data = &state;
-	err = proc_dointvec_minmax(&t, write, buffer, lenp, ppos);
+	err = proc_dointvec_minmax(table, write, buffer, lenp, ppos);
 	if (err < 0)
 		return err;
-	if (write)
-		set_numabalancing_state(state);
+	if (write) {
+		if (sysctl_numa_balancing_mode & NUMA_BALANCING_NORMAL)
+			check_numa_promotion_mode();
+		else if (sysctl_numa_balancing_mode & NUMA_BALANCING_TIERED_MEMORY)
+			numa_promotion_tiered_enabled = true;
+
+		set_numabalancing_state(*(int *)table->data);
+	}
 	return err;
 }
 #endif
diff --git a/kernel/sched/fair.c b/kernel/sched/fair.c
index 210612c9d1e9..45e39832a2b1 100644
--- a/kernel/sched/fair.c
+++ b/kernel/sched/fair.c
@@ -1424,7 +1424,7 @@ bool should_numa_migrate_memory(struct task_struct *p, struct page * page,
 
 	count_vm_numa_event(PGPROMOTE_CANDIDATE);
 
-	if (flags & TNF_DEMOTED)
+	if (numa_demotion_enabled && (flags & TNF_DEMOTED))
 		count_vm_numa_event(PGPROMOTE_CANDIDATE_DEMOTED);
 
 	if (page_is_file_lru(page))
@@ -1435,6 +1435,14 @@ bool should_numa_migrate_memory(struct task_struct *p, struct page * page,
 	this_cpupid = cpu_pid_to_cpupid(dst_cpu, current->pid);
 	last_cpupid = page_cpupid_xchg_last(page, this_cpupid);
 
+	/*
+	 * The pages in non-toptier memory node should be migrated
+	 * according to hot/cold instead of accessing CPU node.
+	 */
+	if (numa_promotion_tiered_enabled && !node_is_toptier(src_nid))
+		return true;
+
+
 	/*
 	 * Allow first faults or private faults to migrate immediately early in
 	 * the lifetime of a task. The magic number 4 is based on waiting for
diff --git a/kernel/sched/sched.h b/kernel/sched/sched.h
index 6057ad67d223..379f3b6f1a3f 100644
--- a/kernel/sched/sched.h
+++ b/kernel/sched/sched.h
@@ -51,6 +51,7 @@
 #include <linux/kthread.h>
 #include <linux/membarrier.h>
 #include <linux/migrate.h>
+#include <linux/mempolicy.h>
 #include <linux/mm_inline.h>
 #include <linux/mmu_context.h>
 #include <linux/nmi.h>
diff --git a/kernel/sysctl.c b/kernel/sysctl.c
index 6b6653529d92..751b52062eb4 100644
--- a/kernel/sysctl.c
+++ b/kernel/sysctl.c
@@ -113,6 +113,7 @@ static int sixty = 60;
 
 static int __maybe_unused neg_one = -1;
 static int __maybe_unused two = 2;
+static int __maybe_unused three = 3;
 static int __maybe_unused four = 4;
 static unsigned long zero_ul;
 static unsigned long one_ul = 1;
@@ -1840,12 +1841,12 @@ static struct ctl_table kern_table[] = {
 	},
 	{
 		.procname	= "numa_balancing",
-		.data		= NULL, /* filled in by handler */
-		.maxlen		= sizeof(unsigned int),
+		.data		= &sysctl_numa_balancing_mode,
+		.maxlen		= sizeof(int),
 		.mode		= 0644,
 		.proc_handler	= sysctl_numa_balancing,
 		.extra1		= SYSCTL_ZERO,
-		.extra2		= SYSCTL_ONE,
+		.extra2		= &three,
 	},
 #endif /* CONFIG_NUMA_BALANCING */
 #endif /* CONFIG_SCHED_DEBUG */
diff --git a/mm/huge_memory.c b/mm/huge_memory.c
index e9d7b9125c5e..b76a0990c5f1 100644
--- a/mm/huge_memory.c
+++ b/mm/huge_memory.c
@@ -22,6 +22,7 @@
 #include <linux/freezer.h>
 #include <linux/pfn_t.h>
 #include <linux/mman.h>
+#include <linux/mempolicy.h>
 #include <linux/memremap.h>
 #include <linux/pagemap.h>
 #include <linux/debugfs.h>
@@ -1849,16 +1850,24 @@ int change_huge_pmd(struct vm_area_struct *vma, pmd_t *pmd,
 	}
 #endif
 
-	/*
-	 * Avoid trapping faults against the zero page. The read-only
-	 * data is likely to be read-cached on the local CPU and
-	 * local/remote hits to the zero page are not interesting.
-	 */
-	if (prot_numa && is_huge_zero_pmd(*pmd))
-		goto unlock;
+	if (prot_numa) {
+		struct page *page;
+		/*
+		 * Avoid trapping faults against the zero page. The read-only
+		 * data is likely to be read-cached on the local CPU and
+		 * local/remote hits to the zero page are not interesting.
+		 */
+		if (is_huge_zero_pmd(*pmd))
+			goto unlock;
 
-	if (prot_numa && pmd_protnone(*pmd))
-		goto unlock;
+		if (pmd_protnone(*pmd))
+			goto unlock;
+
+		/* skip scanning toptier node */
+		page = pmd_page(*pmd);
+		if (numa_promotion_tiered_enabled && node_is_toptier(page_to_nid(page)))
+			goto unlock;
+	}
 
 	/*
 	 * In case prot_numa, we are under mmap_read_lock(mm). It's critical
diff --git a/mm/mprotect.c b/mm/mprotect.c
index 94188df1ee55..3171f435925b 100644
--- a/mm/mprotect.c
+++ b/mm/mprotect.c
@@ -83,6 +83,7 @@ static unsigned long change_pte_range(struct vm_area_struct *vma, pmd_t *pmd,
 			 */
 			if (prot_numa) {
 				struct page *page;
+				int nid;
 
 				/* Avoid TLB flush if possible */
 				if (pte_protnone(oldpte))
@@ -109,7 +110,12 @@ static unsigned long change_pte_range(struct vm_area_struct *vma, pmd_t *pmd,
 				 * Don't mess with PTEs if page is already on the node
 				 * a single-threaded process is running on.
 				 */
-				if (target_node == page_to_nid(page))
+				nid = page_to_nid(page);
+				if (target_node == nid)
+					continue;
+
+				/* skip scanning toptier node */
+				if (numa_promotion_tiered_enabled && node_is_toptier(nid))
 					continue;
 			}
 

-- 
2.30.2


# [PATCH 3/5] Decouple reclaim and allocation for toptier nodes

With a tight memory constraint, we need to proactively keep some
free memory in toptier node, such that 1) new allocation which is
mainly for request processing can be directly put in the toptier
node and 2) toptier node is able to accept hot pages promoted from
non-toptier node. To achieve that, we decouple the reclamation and
allocation mechanism, i.e. reclamation gets triggered at a different
watermark -- WMARK_DEMOTE, while allocation checks for the traditional
WMARK_HIGH. In this way, toptier nodes can maintain some free space to
accept both new allocation and promotion from non-toptier nodes.

On each toptier memory node, kswapd daemon is woken up to demote memory
when free memory on the node falls below the following fraction

    demote_scale_factor/10000

The default value of demote_scale_factor is 200 , (i.e. 2%) so kswapd will
be woken up when available free memory on a node falls below 2%. The
demote_scale_factor can be adjusted higher if we need kswapd to keep more
free memory around by updating the sysctl variable

    /proc/sys/vm/demote_scale_factor

Signed-off-by: Hasan Al Maruf <hasanalmaruf@fb.com>
---
 Documentation/admin-guide/sysctl/vm.rst | 12 +++++++++
 include/linux/mempolicy.h               |  5 ++++
 include/linux/mm.h                      |  4 +++
 include/linux/mmzone.h                  |  5 ++++
 kernel/sched/fair.c                     |  3 +++
 kernel/sysctl.c                         | 12 ++++++++-
 mm/mempolicy.c                          | 23 +++++++++++++++++
 mm/page_alloc.c                         | 34 ++++++++++++++++++++++++-
 mm/vmscan.c                             | 26 +++++++++++++++++++
 mm/vmstat.c                             |  7 ++++-
 10 files changed, 128 insertions(+), 3 deletions(-)

## /Documentation/admin-guide/sysctl/vm.rst
diff --git a/Documentation/admin-guide/sysctl/vm.rst b/Documentation/admin-guide/sysctl/vm.rst
index 586cd4b86428..027b1f31fec1 100644
--- a/Documentation/admin-guide/sysctl/vm.rst
+++ b/Documentation/admin-guide/sysctl/vm.rst
@@ -74,6 +74,7 @@ Currently, these files are in /proc/sys/vm:
 - vfs_cache_pressure
 - watermark_boost_factor
 - watermark_scale_factor
+- demote_scale_factor
 - zone_reclaim_mode
 
 
@@ -961,6 +962,17 @@ that the number of free pages kswapd maintains for latency reasons is
 too small for the allocation bursts occurring in the system. This knob
 can then be used to tune kswapd aggressiveness accordingly.
 
+demote_scale_factor
+===================
+
+This factor controls when kswapd wakes up to demote pages from toptier
+nodes. It defines the amount of memory left in a toptier node/system
+before kswapd is woken up and how much memory needs to be free from those
+nodes before kswapd goes back to sleep.
+
+The unit is in fractions of 10,000. The default value of 200 means if there
+are less than 2% of free toptier memory in a node/system, we will start  to
+demote pages from that node.
 
 zone_reclaim_mode

## /include/linux/mempolicy.h
diff --git a/include/linux/mempolicy.h b/include/linux/mempolicy.h
index ab57b6a82e0a..0a76ac103b17 100644
--- a/include/linux/mempolicy.h
+++ b/include/linux/mempolicy.h
@@ -145,6 +145,7 @@ extern void numa_default_policy(void);
 extern void numa_policy_init(void);
 extern void mpol_rebind_task(struct task_struct *tsk, const nodemask_t *new);
 extern void mpol_rebind_mm(struct mm_struct *mm, nodemask_t *new);
+extern void check_toptier_balanced(void);
 
 extern int huge_node(struct vm_area_struct *vma,
 				unsigned long addr, gfp_t gfp_flags,
@@ -299,6 +300,10 @@ static inline nodemask_t *policy_nodemask_current(gfp_t gfp)
 	return NULL;
 }
 
+static inline void check_toptier_balanced(void)
+{
+}
+
 #define numa_demotion_enabled	false
 #define numa_promotion_tiered_enabled	false
 #endif /* CONFIG_NUMA */
diff --git a/include/linux/mm.h b/include/linux/mm.h
index 9a226787464e..4748e57b7c68 100644
--- a/include/linux/mm.h
+++ b/include/linux/mm.h
@@ -3153,6 +3153,10 @@ static inline bool debug_guardpage_enabled(void) { return false; }
 static inline bool page_is_guard(struct page *page) { return false; }
 #endif /* CONFIG_DEBUG_PAGEALLOC */
 
+#ifdef CONFIG_MIGRATION
+extern int demote_scale_factor;
+#endif
+
 #if MAX_NUMNODES > 1
 void __init setup_nr_node_ids(void);
 #else
diff --git a/include/linux/mmzone.h b/include/linux/mmzone.h
index 47946cec7584..070284feac03 100644
--- a/include/linux/mmzone.h
+++ b/include/linux/mmzone.h
@@ -329,12 +329,14 @@ enum zone_watermarks {
 	WMARK_MIN,
 	WMARK_LOW,
 	WMARK_HIGH,
+	WMARK_DEMOTE,
 	NR_WMARK
 };
 
 #define min_wmark_pages(z) (z->_watermark[WMARK_MIN] + z->watermark_boost)
 #define low_wmark_pages(z) (z->_watermark[WMARK_LOW] + z->watermark_boost)
 #define high_wmark_pages(z) (z->_watermark[WMARK_HIGH] + z->watermark_boost)
+#define demote_wmark_pages(z) (z->_watermark[WMARK_DEMOTE] + z->watermark_boost)
 #define wmark_pages(z, i) (z->_watermark[i] + z->watermark_boost)
 
 struct per_cpu_pages {
@@ -884,6 +886,7 @@ bool zone_watermark_ok(struct zone *z, unsigned int order,
 		unsigned int alloc_flags);
 bool zone_watermark_ok_safe(struct zone *z, unsigned int order,
 		unsigned long mark, int highest_zoneidx);
+bool pgdat_toptier_balanced(pg_data_t *pgdat, int order, int zone_idx);
 /*
  * Memory initialization context, use to differentiate memory added by
  * the platform statically or via memory hotplug interface.
@@ -1011,6 +1014,8 @@ int min_free_kbytes_sysctl_handler(struct ctl_table *, int, void *, size_t *,
 		loff_t *);
 int watermark_scale_factor_sysctl_handler(struct ctl_table *, int, void *,
 		size_t *, loff_t *);
+int demote_scale_factor_sysctl_handler(struct ctl_table *, int, void __user *,
+		size_t *, loff_t *);
 extern int sysctl_lowmem_reserve_ratio[MAX_NR_ZONES];
 int lowmem_reserve_ratio_sysctl_handler(struct ctl_table *, int, void *,
 		size_t *, loff_t *);
diff --git a/kernel/sched/fair.c b/kernel/sched/fair.c
index 45e39832a2b1..6cada31f7265 100644
--- a/kernel/sched/fair.c
+++ b/kernel/sched/fair.c
@@ -21,6 +21,8 @@
  *  Copyright (C) 2007 Red Hat, Inc., Peter Zijlstra
  */
 #include "sched.h"
+#include <trace/events/sched.h>
+#include <linux/mempolicy.h>
 
 /*
  * Targeted preemption latency for CPU-bound tasks:
@@ -10802,6 +10804,7 @@ void trigger_load_balance(struct rq *rq)
 		raise_softirq(SCHED_SOFTIRQ);
 
 	nohz_balancer_kick(rq);
+	check_toptier_balanced();
 }
 
 static void rq_online_fair(struct rq *rq)
## /kernel/sysctl.c
diff --git a/kernel/sysctl.c b/kernel/sysctl.c
index 751b52062eb4..7d2995045a94 100644
--- a/kernel/sysctl.c
+++ b/kernel/sysctl.c
@@ -112,6 +112,7 @@ static int sixty = 60;
 #endif
 
 static int __maybe_unused neg_one = -1;
+static int __maybe_unused one = 1;
 static int __maybe_unused two = 2;
 static int __maybe_unused three = 3;
 static int __maybe_unused four = 4;
@@ -121,8 +122,8 @@ static unsigned long long_max = LONG_MAX;
 static int one_hundred = 100;
 static int two_hundred = 200;
 static int one_thousand = 1000;
-#ifdef CONFIG_PRINTK
 static int ten_thousand = 10000;
+#ifdef CONFIG_PRINTK
 #endif
 #ifdef CONFIG_PERF_EVENTS
 static int six_hundred_forty_kb = 640 * 1024;
@@ -3000,6 +3001,15 @@ static struct ctl_table vm_table[] = {
 		.extra1		= SYSCTL_ONE,
 		.extra2		= &one_thousand,
 	},
+	{
+		.procname       = "demote_scale_factor",
+		.data           = &demote_scale_factor,
+		.maxlen         = sizeof(demote_scale_factor),
+		.mode           = 0644,
+		.proc_handler   = demote_scale_factor_sysctl_handler,
+		.extra1         = &one,
+		.extra2         = &ten_thousand,
+	},
 	{
 		.procname	= "percpu_pagelist_fraction",
 		.data		= &percpu_pagelist_fraction,
diff --git a/mm/mempolicy.c b/mm/mempolicy.c
index 580e76ae58e6..ba9b1322bd48 100644
--- a/mm/mempolicy.c
+++ b/mm/mempolicy.c
@@ -1042,6 +1042,29 @@ static long do_get_mempolicy(int *policy, nodemask_t *nmask,
 	return err;
 }
 
+void check_toptier_balanced(void)
+{
+	int nid;
+	int balanced;
+
+	if (!numa_promotion_tiered_enabled)
+		return;
+
+	for_each_node_state(nid, N_MEMORY) {
+		pg_data_t *pgdat = NODE_DATA(nid);
+
+		if (!node_is_toptier(nid))
+			continue;
+
+		balanced = pgdat_toptier_balanced(pgdat, 0, ZONE_MOVABLE);
+		if (!balanced) {
+			pgdat->kswapd_order = 0;
+			pgdat->kswapd_highest_zoneidx = ZONE_NORMAL;
+			wakeup_kswapd(pgdat->node_zones + ZONE_NORMAL, 0, 1, ZONE_NORMAL);
+		}
+	}
+}
+
 #ifdef CONFIG_MIGRATION
 /*
  * page migration, thp tail pages can be passed.
diff --git a/mm/page_alloc.c b/mm/page_alloc.c
index 5f1dd104cf8e..8638e24e1b2f 100644
--- a/mm/page_alloc.c
+++ b/mm/page_alloc.c
@@ -3599,7 +3599,8 @@ struct page *rmqueue(struct zone *preferred_zone,
 	if (test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags)) {
 		clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
 		wakeup_kswapd(zone, 0, 0, zone_idx(zone));
-	}
+	} else if (!pgdat_toptier_balanced(zone->zone_pgdat, order, zone_idx(zone)))
+		wakeup_kswapd(zone, 0, 0, zone_idx(zone));
 
 	VM_BUG_ON_PAGE(page && bad_range(zone, page), page);
 	return page;
@@ -8047,6 +8048,22 @@ static void __setup_per_zone_wmarks(void)
 		zone->_watermark[WMARK_LOW]  = min_wmark_pages(zone) + tmp;
 		zone->_watermark[WMARK_HIGH] = min_wmark_pages(zone) + tmp * 2;
 
+		if (numa_promotion_tiered_enabled) {
+			tmp = mult_frac(zone_managed_pages(zone), demote_scale_factor, 10000);
+
+			/*
+			 * Clamp demote watermark between twice high watermark
+			 * and max managed pages.
+			 */
+			if (tmp < 2 * zone->_watermark[WMARK_HIGH])
+				tmp = 2 * zone->_watermark[WMARK_HIGH];
+			if (tmp > zone_managed_pages(zone))
+				tmp = zone_managed_pages(zone);
+			zone->_watermark[WMARK_DEMOTE] = tmp;
+
+			zone->watermark_boost = 0;
+		}
+
 		spin_unlock_irqrestore(&zone->lock, flags);
 	}
 
@@ -8163,6 +8180,21 @@ int watermark_scale_factor_sysctl_handler(struct ctl_table *table, int write,
 	return 0;
 }
 
+int demote_scale_factor_sysctl_handler(struct ctl_table *table, int write,
+	void __user *buffer, size_t *length, loff_t *ppos)
+{
+	int rc;
+
+	rc = proc_dointvec_minmax(table, write, buffer, length, ppos);
+	if (rc)
+		return rc;
+
+	if (write)
+		setup_per_zone_wmarks();
+
+	return 0;
+}
+
 #ifdef CONFIG_NUMA
 static void setup_min_unmapped_ratio(void)
 {
## /mm/vmscan.c

@@ -41,6 +41,7 @@
 #include <linux/kthread.h>
 #include <linux/freezer.h>
 #include <linux/memcontrol.h>
+#include <linux/mempolicy.h>
 #include <linux/migrate.h>
 #include <linux/delayacct.h>
 #include <linux/sysctl.h>
@@ -190,6 +191,7 @@ static void set_task_reclaim_state(struct task_struct *task,
 
 static LIST_HEAD(shrinker_list);
 static DECLARE_RWSEM(shrinker_rwsem);
+int demote_scale_factor = 200;
 
 #ifdef CONFIG_MEMCG
 /*
@@ -3598,6 +3600,30 @@ static bool pgdat_balanced(pg_data_t *pgdat, int order, int highest_zoneidx)
 	return false;
 }
 
+bool pgdat_toptier_balanced(pg_data_t *pgdat, int order, int zone_idx)
+{
+	unsigned long mark;
+	struct zone *zone;
+
+	if (!node_is_toptier(pgdat->node_id) ||
+		!numa_promotion_tiered_enabled ||
+		order > 0 || zone_idx < ZONE_NORMAL) {
+		return true;
+	}
+
+	zone = pgdat->node_zones + ZONE_NORMAL;
+
+	if (!managed_zone(zone))
+		return true;
+
+	mark = min(demote_wmark_pages(zone), zone_managed_pages(zone));
+
+	if (zone_page_state(zone, NR_FREE_PAGES) < mark)
+		return false;
+
+	return true;
+}
+
 /* Clear pgdat state for congested, dirty or under writeback. */
 static void clear_pgdat_congested(pg_data_t *pgdat)
 {
## /mm/vmstat.c
index cda2505bb21f..4309f79a6132 100644
--- a/mm/vmstat.c
+++ b/mm/vmstat.c
@@ -28,6 +28,7 @@
 #include <linux/mm_inline.h>
 #include <linux/page_ext.h>
 #include <linux/page_owner.h>
+#include <linux/migrate.h>
 
 #include "internal.h"
 
@@ -1649,7 +1650,9 @@ static void zoneinfo_show_print(struct seq_file *m, pg_data_t *pgdat,
 							struct zone *zone)
 {
 	int i;
-	seq_printf(m, "Node %d, zone %8s", pgdat->node_id, zone->name);
+	seq_printf(m, "Node %d, zone %8s, toptier %d next_demotion_node %d",
+			pgdat->node_id, zone->name, node_is_toptier(pgdat->node_id),
+			next_demotion_node(pgdat->node_id));
 	if (is_zone_first_populated(pgdat, zone)) {
 		seq_printf(m, "\n  per-node stats");
 		for (i = 0; i < NR_VM_NODE_STAT_ITEMS; i++) {
@@ -1666,6 +1669,7 @@ static void zoneinfo_show_print(struct seq_file *m, pg_data_t *pgdat,
 		   "\n        min      %lu"
 		   "\n        low      %lu"
 		   "\n        high     %lu"
+		   "\n        demote   %lu"
 		   "\n        spanned  %lu"
 		   "\n        present  %lu"
 		   "\n        managed  %lu"
@@ -1674,6 +1678,7 @@ static void zoneinfo_show_print(struct seq_file *m, pg_data_t *pgdat,
 		   min_wmark_pages(zone),
 		   low_wmark_pages(zone),
 		   high_wmark_pages(zone),
+		   node_is_toptier(pgdat->node_id) ? demote_wmark_pages(zone) : 0,
 		   zone->spanned_pages,
 		   zone->present_pages,
 		   zone_managed_pages(zone),

-- 
2.30.2


# [PATCH 4/5] Reclaim to satisfy WMARK_DEMOTE on toptier nodes


When kswapd is wakenup on a toptier node in a tiered-memory NUMA
balancing mode, it reclaims pages until the toptier node is balanced
and the number of free pages on toptier node satisfies WMARK_DEMOTE.

When THP (Transparent Huge Page) is enabled, sometimes demotion/promotion
between the memory nodes may pause for several hundreds of seconds as
the pages in the toptier node may sometimes become so hot, that kswapd
fails to reclaim any page.  Finally, the kswapd failure count
(pgdat->kswapd_failures) reaches its max value and kswapd will not be
waken up until a successful direct reclaiming. For general use case,
this isn't a big problem as the memory users will do direct reclaim
finally and trigger successful direct reclaiming or OOM to fix the
issue. But in memory tiering system, the demotion and promotion will
avoid to create too much memory pressure on the fast memory node, so
direct reclaiming will not be triggered to resolve the issue. To
resolve this, when promotion enabled, kswapd will be waken up every
10 seconds to try to free some pages to recover kswapd failures.

## 修改文件
 mm/vmscan.c | 42 ++++++++++++++++++++++++++++++++++++------
 1 file changed, 36 insertions(+), 6 deletions(-)

## /mm/vmscan.c文件
diff --git a/mm/vmscan.c b/mm/vmscan.c
index c39b217effa9..1e87221f2b58 100644
--- a/mm/vmscan.c
+++ b/mm/vmscan.c
### get_scan_count
@@ -2386,8 +2386,14 @@ static void get_scan_count(struct lruvec *lruvec, struct scan_control *sc,
 	unsigned long ap, fp;
 	enum lru_list lru;
 
-	/* If we have no swap space, do not bother scanning anon pages. */
-	if (!sc->may_swap || !can_reclaim_anon_pages(memcg, pgdat->node_id, sc)) {
+	/*
+	 * If we have no swap space, do not bother scanning anon pages.
+	 * However, anon pages on toptier node can be demoted via reclaim
+	 * when numa promotion is enabled. Disable the check to prevent
+	 * demotion for no swap space when numa promotion is enabled.
+	 */
+	if (!numa_promotion_tiered_enabled &&
+		(!sc->may_swap || !can_reclaim_anon_pages(memcg, pgdat->node_id, sc))) {
 		scan_balance = SCAN_FILE;
 		goto out;
 	}
### shrink_node
@@ -2916,7 +2922,10 @@ static void shrink_node(pg_data_t *pgdat, struct scan_control *sc)
 			if (!managed_zone(zone))
 				continue;
 
-			total_high_wmark += high_wmark_pages(zone);
+			if (numa_promotion_tiered_enabled && node_is_toptier(pgdat->node_id))
+				total_high_wmark += demote_wmark_pages(zone);
+			else
+				total_high_wmark += high_wmark_pages(zone);
 		}
 
 		/*

@@ -3574,6 +3583,9 @@ static bool pgdat_balanced(pg_data_t *pgdat, int order, int highest_zoneidx)
 	unsigned long mark = -1;
 	struct zone *zone;
 
+	if (numa_promotion_tiered_enabled && node_is_toptier(pgdat->node_id) &&
+			highest_zoneidx >= ZONE_NORMAL)
+		return pgdat_toptier_balanced(pgdat, 0, highest_zoneidx);
 	/*
 	 * Check watermarks bottom-up as lower zones are more likely to
 	 * meet watermarks.
@@ -3692,7 +3704,10 @@ static bool kswapd_shrink_node(pg_data_t *pgdat,
 		if (!managed_zone(zone))
 			continue;
 
-		sc->nr_to_reclaim += max(high_wmark_pages(zone), SWAP_CLUSTER_MAX);
+		if (numa_promotion_tiered_enabled && node_is_toptier(pgdat->node_id))
+			sc->nr_to_reclaim += max(demote_wmark_pages(zone), SWAP_CLUSTER_MAX);
+		else
+			sc->nr_to_reclaim += max(high_wmark_pages(zone), SWAP_CLUSTER_MAX);
 	}
 
 	/*
@@ -4021,8 +4036,23 @@ static void kswapd_try_to_sleep(pg_data_t *pgdat, int alloc_order, int reclaim_o
 		 */
 		set_pgdat_percpu_threshold(pgdat, calculate_normal_threshold);
 
-		if (!kthread_should_stop())
-			schedule();
+		if (!kthread_should_stop()) {
+			/*
+			 * In numa promotion modes, try harder to recover from
+			 * kswapd failures, because direct reclaiming may be
+			 * not triggered.
+			 */
+			if (numa_promotion_tiered_enabled &&
+						node_is_toptier(pgdat->node_id) &&
+					pgdat->kswapd_failures >= MAX_RECLAIM_RETRIES) {
+				remaining = schedule_timeout(10 * HZ);
+				if (!remaining) {
+					pgdat->kswapd_highest_zoneidx = ZONE_MOVABLE;
+					pgdat->kswapd_order = 0;
+				}
+			} else
+				schedule();
+		}
 
 		set_pgdat_percpu_threshold(pgdat, calculate_pressure_threshold);
 	} else {

-- 
2.30.2



# [PATCH 5/5] active LRU-based promotion to avoid ping-pong


Whenever a remote hint-fault happens on a page, the default NUMA
balancing promotes the page without checking its state. As a result,
cold pages with very infrequent accesses, can still be the promotion
candidates. Once promoted to the local node, these type of pages may
shortly become the demotion candidate if the toptier nodes are always
under pressure. Thus, promotion traffics generated from infrequently
accessed pages can easily fill up the toptier node's reclaimed free
spaces and eventually generate a higher demotion traffic for the non-
toptier node. This demotion-promotion ping-pong causes unnecessary
traffic over the memory nodes and can negatively impact on the
performance of memory bound applications.

To solve this ping-pong issue, instead of instant promotion, we check
a page's age through its position in the LRU list. If the faulted page
is in inactive LRU, then we don't instantly consider the page as the
promotion candidate as it might be an infrequently accessed pages. We
only consider the faulted pages that are in the active LRUs (either
of anon or file active LRU) as the promotion candidate. This approach
significantly reduce the promotion traffic and always maintain a
satisfactory amount of free memory on the toptier node to support both
new allocations and promotion from non-tortier nodes.

Signed-off-by: Hasan Al Maruf <hasanalmaruf@fb.com>
---
 mm/memory.c | 13 +++++++++++++
 1 file changed, 13 insertions(+)

diff --git a/mm/memory.c b/mm/memory.c
index 314fe3b2f462..1c76f074784a 100644
--- a/mm/memory.c
+++ b/mm/memory.c
@@ -4202,6 +4202,19 @@ static vm_fault_t do_numa_page(struct vm_fault *vmf)
 
 	last_cpupid = page_cpupid_last(page);
 	page_nid = page_to_nid(page);
+
+	/* Only migrate pages that are active on non-toptier node */
+	if (numa_promotion_tiered_enabled &&
+		!node_is_toptier(page_nid) &&
+		!PageActive(page)) {
+		count_vm_numa_event(NUMA_HINT_FAULTS);
+		if (page_nid == numa_node_id())
+			count_vm_numa_event(NUMA_HINT_FAULTS_LOCAL);
+		mark_page_accessed(page);
+		pte_unmap_unlock(vmf->pte, vmf->ptl);
+		goto out;
+	}
+
 	target_nid = numa_migrate_prep(page, vma, vmf->address, page_nid,
 			&flags);
 	pte_unmap_unlock(vmf->pte, vmf->ptl);

-- 
2.30.2

