pgd_alloc
========================================

path: arch/arm/mm/pgd.c
```
/*
 * need to get a 16k page for level 1
 */
pgd_t *pgd_alloc(struct mm_struct *mm)
{
    pgd_t *new_pgd, *init_pgd;
    pud_t *new_pud, *init_pud;
    pmd_t *new_pmd, *init_pmd;
    pte_t *new_pte, *init_pte;

    new_pgd = __pgd_alloc();
    if (!new_pgd)
        goto no_pgd;

    memset(new_pgd, 0, USER_PTRS_PER_PGD * sizeof(pgd_t));

    /*
     * Copy over the kernel and IO PGD entries
     */
    init_pgd = pgd_offset_k(0);
    memcpy(new_pgd + USER_PTRS_PER_PGD, init_pgd + USER_PTRS_PER_PGD,
               (PTRS_PER_PGD - USER_PTRS_PER_PGD) * sizeof(pgd_t));

    clean_dcache_area(new_pgd, PTRS_PER_PGD * sizeof(pgd_t));

#ifdef CONFIG_ARM_LPAE
    /*
     * Allocate PMD table for modules and pkmap mappings.
     */
    new_pud = pud_alloc(mm, new_pgd + pgd_index(MODULES_VADDR),
                MODULES_VADDR);
    if (!new_pud)
        goto no_pud;

    new_pmd = pmd_alloc(mm, new_pud, 0);
    if (!new_pmd)
        goto no_pmd;
#endif

    if (!vectors_high()) {
        /*
         * On ARM, first page must always be allocated since it
         * contains the machine vectors. The vectors are always high
         * with LPAE.
         */
        new_pud = pud_alloc(mm, new_pgd, 0);
        if (!new_pud)
            goto no_pud;

        new_pmd = pmd_alloc(mm, new_pud, 0);
        if (!new_pmd)
            goto no_pmd;

        new_pte = pte_alloc_map(mm, NULL, new_pmd, 0);
        if (!new_pte)
            goto no_pte;

#ifndef CONFIG_ARM_LPAE
        /*
         * Modify the PTE pointer to have the correct domain.  This
         * needs to be the vectors domain to avoid the low vectors
         * being unmapped.
         */
        pmd_val(*new_pmd) &= ~PMD_DOMAIN_MASK;
        pmd_val(*new_pmd) |= PMD_DOMAIN(DOMAIN_VECTORS);
#endif

        init_pud = pud_offset(init_pgd, 0);
        init_pmd = pmd_offset(init_pud, 0);
        init_pte = pte_offset_map(init_pmd, 0);
        set_pte_ext(new_pte + 0, init_pte[0], 0);
        set_pte_ext(new_pte + 1, init_pte[1], 0);
        pte_unmap(init_pte);
        pte_unmap(new_pte);
    }

    return new_pgd;

no_pte:
    pmd_free(mm, new_pmd);
    mm_dec_nr_pmds(mm);
no_pmd:
    pud_free(mm, new_pud);
no_pud:
    __pgd_free(new_pgd);
no_pgd:
    return NULL;
}
```