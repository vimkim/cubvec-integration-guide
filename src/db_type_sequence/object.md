# Object Directory

```cpp
schema_template.c
539-
540-/* DOMAIN DECODING */
541-/*
542- * resolve_class_domain()
543- * get_domain_internal()
544- * get_domain() - Maps a domain string into a domain structure.
545- */
546-
547-static int
548-resolve_class_domain (SM_TEMPLATE * tmp, DB_DOMAIN * domain)
549-{
550-  int error = NO_ERROR;
551-  DB_DOMAIN *tmp_domain;
552-
553-  if (domain)
554-    {
555-      switch (TP_DOMAIN_TYPE (domain))
556- {
557- case DB_TYPE_SET:
558- case DB_TYPE_MULTISET:
559: case DB_TYPE_SEQUENCE:
560-   tmp_domain = domain->setdomain;
561-   while (tmp_domain)
562-     {
563-       error = resolve_class_domain (tmp, tmp_domain);
564-       if (error != NO_ERROR)
565-  {
566-    return error;
567-  }
568-       tmp_domain = tmp_domain->next;
569-     }
570-   break;
571-
572- case DB_TYPE_OBJECT:
573-   if (domain->self_ref)
574-     {
575-       domain->type = tp_Type_null;
576-       /* kludge, store the template as the "class" for this special domain */
577-       domain->class_mop = (MOP) tmp;
578-     }
579-   break;
```

```cpp
transform_cl.c
1666-
1667-  /* store NULL for empty set */
1668-  if (list == NULL)
1669-    return ER_FAILED;
1670-
1671-  for (count = 0, l = list; l != NULL; l = l->next)
1672-    {
1673-      if (WS_IS_DELETED (l->op))
1674- {
1675-   continue;
1676- }
1677-      count++;
1678-    }
1679-
1680-  if (count == 0)
1681-    {
1682-      return NO_ERROR;
1683-    }
1684-
1685-  /* w/ domain, no bound bits, no offsets, no tags, no substructure header */
1686:  or_put_set_header (buf, DB_TYPE_SEQUENCE, count, 1, 0, 0, 0, 0);
1687-
1688-  /* use the generic "object" domain */
1689-  or_put_int (buf, OR_INT_SIZE); /* size of the domain */
1690-  or_put_domain (buf, &tp_Object_domain, 0, 0); /* actual domain */
1691-
1692-  /* should be using something other than or_pack_mop here ! */
1693-  for (l = list; l != NULL; l = l->next)
1694-    {
1695-      if (WS_IS_DELETED (l->op))
1696- {
1697-   continue;
1698- }
1699-      or_pack_mop (buf, l->op);
1700-    }
1701-
1702-  return NO_ERROR;
1703-}
1704-
1705-/*
1706- * get_object_set - Extracts a sequence of objects in a disk representation



1807- */
1808-static void
1809-put_substructure_set (OR_BUF * buf, DB_LIST * list, LWRITER writer, OID * class_, int repid)
1810-{
1811-  DB_LIST *l;
1812-  char *start;
1813-  int count;
1814-  char *offset_ptr;
1815-
1816-  /* store NULL for empty list */
1817-  if (list == NULL)
1818-    {
1819-      return;
1820-    }
1821-
1822-  count = 0;
1823-  for (l = list; l != NULL; l = l->next, count++);
1824-
1825-  /* with domain, no bound bits, with offsets, no tags, common sub header */
1826-  start = buf->ptr;
1827:  or_put_set_header (buf, DB_TYPE_SEQUENCE, count, 1, 0, 1, 0, 1);
1828-
1829-  if (!count)
1830-    {
1831-      /* we must at least store the domain even though there will be no elements */
1832-      or_put_int (buf, OR_SUB_DOMAIN_SIZE);
1833-      or_put_sub_domain (buf);
1834-    }
1835-  else
1836-    {
1837-      /* begin an offset table */
1838-      offset_ptr = buf->ptr;
1839-      or_advance (buf, OR_VAR_TABLE_SIZE (count));
1840-
1841-      /* write the domain */
1842-      or_put_int (buf, OR_SUB_DOMAIN_SIZE);
1843-      or_put_sub_domain (buf);
1844-
1845-      /* write the common substructure header */
1846-      or_put_oid (buf, class_);
1847-      or_put_int (buf, repid);



3079-      if (triggers != NULL)
3080- {
3081-   att->triggers = tr_make_schema_cache (TR_CACHE_ATTRIBUTE, triggers);
3082- }
3083-      /* variable attribute 5: property list */
3084-      /* formerly disk_to_att_extension(buf, att, vars[5].length); */
3085-      att->properties = get_property_list (buf, vars[ORC_ATT_PROPERTIES_INDEX].length);
3086-
3087-      classobj_initialize_default_expr (&att->default_value.default_expr);
3088-      att->on_update_default_expr = DB_DEFAULT_NONE;
3089-      if (att->properties)
3090- {
3091-   if (classobj_get_prop (att->properties, "update_default", &value) > 0)
3092-     {
3093-       att->on_update_default_expr = (DB_DEFAULT_EXPR_TYPE) db_get_int (&value);
3094-     }
3095-
3096-   if (classobj_get_prop (att->properties, "default_expr", &value) > 0)
3097-     {
3098-       /* We have two cases: simple and complex expressions. */
3099:       if (DB_VALUE_TYPE (&value) == DB_TYPE_SEQUENCE)
3100-  {
3101-    DB_SEQ *def_expr_seq;
3102-    DB_VALUE def_expr_op, def_expr_type, def_expr_format;
3103-    const char *def_expr_format_str;
3104-
3105-    assert (set_size (db_get_set (&value)) == 3);
3106-
3107-    def_expr_seq = db_get_set (&value);
3108-
3109-    /* get default expression operator (op of expr) */
3110-    if (set_get_element_nocopy (def_expr_seq, 0, &def_expr_op) != NO_ERROR)
3111-      {
3112-        assert (false);
3113-      }
3114-    assert (DB_VALUE_TYPE (&def_expr_op) == DB_TYPE_INTEGER
3115-     && db_get_int (&def_expr_op) == (int) T_TO_CHAR);
3116-    att->default_value.default_expr.default_expr_op = db_get_int (&def_expr_op);
3117-
3118-    /* get default expression type (arg1 of expr) */
3119-    if (set_get_element_nocopy (def_expr_seq, 1, &def_expr_type) != NO_ERROR)

```

```cpp
object_domain.c
102-  TP_IMPLICIT_COERCION,
103-  TP_FORCE_COERCION
104-} TP_COERCION_MODE;
105-
106-/*
107- * These are arranged to get relative types for symmetrical
108- * coercion selection. The absolute position is not critical.
109- * If two types are mutually coercible, the more general
110- * should appear later. Eg. Float should appear after integer.
111- */
112-
113-static const DB_TYPE db_type_rank[] = { DB_TYPE_NULL,
114-  DB_TYPE_SHORT,
115-  DB_TYPE_INTEGER,
116-  DB_TYPE_BIGINT,
117-  DB_TYPE_NUMERIC,
118-  DB_TYPE_FLOAT,
119-  DB_TYPE_DOUBLE,
120-  DB_TYPE_MONETARY,
121-  DB_TYPE_SET,
122:  DB_TYPE_SEQUENCE,
123-  DB_TYPE_MULTISET,
124-  DB_TYPE_TIME,
125-  DB_TYPE_DATE,
126-  DB_TYPE_TIMESTAMP,
127-  DB_TYPE_TIMESTAMPLTZ,
128-  DB_TYPE_TIMESTAMPTZ,
129-  DB_TYPE_DATETIME,
130-  DB_TYPE_DATETIMELTZ,
131-  DB_TYPE_DATETIMETZ,
132-  DB_TYPE_OID,
133-  DB_TYPE_VOBJ,
134-  DB_TYPE_OBJECT,
135-  DB_TYPE_CHAR,
136-  DB_TYPE_VARCHAR,
137-  DB_TYPE_NCHAR,
138-  DB_TYPE_VARNCHAR,
139-  DB_TYPE_BIT,
140-  DB_TYPE_VARBIT,
141-  DB_TYPE_ELO,
142-  DB_TYPE_BLOB,



1596-   if (dom1->class_mop == NULL)
1597-     {
1598-       match = OID_EQ (&dom1->class_oid, WS_OID (dom2->class_mop));
1599-     }
1600-   else
1601-     {
1602-       match = OID_EQ (WS_OID (dom1->class_mop), &dom2->class_oid);
1603-     }
1604- }
1605-#endif /* defined (SERVER_MODE) */
1606-
1607-      if (match == 0 && exact == TP_SET_MATCH && dom1->class_mop == NULL && OID_ISNULL (&dom1->class_oid))
1608- {
1609-   match = 1;
1610- }
1611-      break;
1612-
1613-    case DB_TYPE_VARIABLE:
1614-    case DB_TYPE_SET:
1615-    case DB_TYPE_MULTISET:
1616:    case DB_TYPE_SEQUENCE:
1617-#if 1
1618-      /* >>>>> NEED MORE CONSIDERATION <<<<< do not check order must be rollback with tp_domain_add() */
1619-      if (dom1->setdomain == dom2->setdomain)
1620- {
1621-   match = 1;
1622- }
1623-      else
1624- {
1625-   int dsize;
1626-
1627-   /* don't bother comparing the lists unless the sizes are the same */
1628-   dsize = tp_domain_size (dom1->setdomain);
1629-   if (dsize == tp_domain_size (dom2->setdomain))
1630-     {
1631-       /* handle the simple single domain case quickly */
1632-       if (dsize == 1)
1633-  {
1634-    match = tp_domain_match (dom1->setdomain, dom2->setdomain, exact);
1635-  }
1636-       else



2030-       if (domain->is_desc == transient->is_desc)
2031-  {
2032-    match = 1;
2033-  }
2034-     }
2035-
2036-   if (match)
2037-     {
2038-       break;
2039-     }
2040-
2041-   *ins_pos = domain;
2042-   domain = domain->next_list;
2043- }
2044-#endif /* !defined (SERVER_MODE) */
2045-      break;
2046-
2047-    case DB_TYPE_VARIABLE:
2048-    case DB_TYPE_SET:
2049-    case DB_TYPE_MULTISET:
2050:    case DB_TYPE_SEQUENCE:
2051-      {
2052- int dsize2;
2053-
2054- dsize2 = tp_domain_size (transient->setdomain);
2055- while (domain)
2056-   {
2057-#if 1
2058-     /* >>>>> NEED MORE CONSIDERATION <<<<< do not check order must be rollback with tp_domain_add() */
2059-     if (domain->setdomain == transient->setdomain)
2060-       {
2061-  match = 1;
2062-       }
2063-     else
2064-       {
2065-  int dsize1;
2066-
2067-  /*
2068-   * don't bother comparing the lists unless the sizes are the
2069-   * same
2070-   */



2826-
2827-      if (dom->is_desc == is_desc)
2828- {
2829-   /* don't bother comparing the lists unless the sizes are the same */
2830-   dsize = tp_setdomain_size (dom);
2831-   if (dsize == src_dsize)
2832-     {
2833-       /* handle the simple single domain case quickly */
2834-       if (dsize == 1)
2835-  {
2836-    if (tp_domain_match (dom->setdomain, setdomain, TP_EXACT_MATCH))
2837-      {
2838-        break;
2839-      }
2840-  }
2841-       else
2842-  {
2843-    TP_DOMAIN *d1, *d2;
2844-    int match, i;
2845-
2846:    if (type == DB_TYPE_SEQUENCE || type == DB_TYPE_MIDXKEY)
2847-      {
2848-        if (dsize == src_dsize)
2849-   {
2850-     match = 1;
2851-     d1 = dom->setdomain;
2852-     d2 = setdomain;
2853-
2854-     for (i = 0; i < dsize; i++)
2855-       {
2856-         match = tp_domain_match (d1, d2, TP_EXACT_MATCH);
2857-         if (match == 0)
2858-    {
2859-      break;
2860-    }
2861-         d1 = d1->next;
2862-         d2 = d2->next;
2863-       }
2864-     if (match == 1)
2865-       {
2866-         break;



3443-    return NULL;
3444-  }
3445-     }
3446-   else
3447-     {
3448-       domain->json_validator = NULL;
3449-     }
3450-
3451-   if (dbuf == NULL)
3452-     {
3453-       domain = tp_domain_cache (domain);
3454-     }
3455-   break;
3456-
3457-   /*
3458-    * things handled in logic outside the switch, shuts up compiler
3459-    * warnings
3460-    */
3461- case DB_TYPE_SET:
3462- case DB_TYPE_MULTISET:
3463: case DB_TYPE_SEQUENCE:
3464- case DB_TYPE_MIDXKEY:
3465-   break;
3466- case DB_TYPE_TABLE:
3467-   break;
3468- case DB_TYPE_ELO:
3469-   assert (false);
3470-   break;
3471- }
3472-    }
3473-
3474-  return domain;
3475-}
3476-
3477-#if defined(ENABLE_UNUSED_FUNCTION)
3478-/*
3479- * tp_create_domain_resolve_value - adjust domain of a DB_VALUE with respect to
3480- * the primitive value of the value
3481- *    return: domain
3482- *    val(in): DB_VALUE
3483- *    domain(in): domain



3585-     case DB_TYPE_CLOB:
3586-     case DB_TYPE_TIME:
3587-     case DB_TYPE_TIMESTAMP:
3588-     case DB_TYPE_TIMESTAMPTZ:
3589-     case DB_TYPE_TIMESTAMPLTZ:
3590-     case DB_TYPE_DATE:
3591-     case DB_TYPE_DATETIME:
3592-     case DB_TYPE_DATETIMETZ:
3593-     case DB_TYPE_DATETIMELTZ:
3594-     case DB_TYPE_MONETARY:
3595-     case DB_TYPE_SUB:
3596-     case DB_TYPE_POINTER:
3597-     case DB_TYPE_ERROR:
3598-     case DB_TYPE_SHORT:
3599-     case DB_TYPE_VOBJ:
3600-     case DB_TYPE_OID:
3601-     case DB_TYPE_NULL:
3602-     case DB_TYPE_VARIABLE:
3603-     case DB_TYPE_SET:
3604-     case DB_TYPE_MULTISET:
3605:     case DB_TYPE_SEQUENCE:
3606-     case DB_TYPE_DB_VALUE:
3607-     case DB_TYPE_VARCHAR:
3608-     case DB_TYPE_VARNCHAR:
3609-     case DB_TYPE_VARBIT:
3610-       found = d;
3611-       break;
3612-
3613-     case DB_TYPE_NUMERIC:
3614-       if ((d->precision == domain->precision) && (d->scale == domain->scale))
3615-  {
3616-    found = d;
3617-  }
3618-       break;
3619-
3620-     case DB_TYPE_CHAR:
3621-     case DB_TYPE_NCHAR:
3622-     case DB_TYPE_BIT:
3623-       /*
3624-        * PR)  1.deficient character related with CHAR & VARCHAR in set.
3625-        * ==> distinguishing VARCHAR from CHAR.



3729-     case DB_TYPE_CLOB:
3730-     case DB_TYPE_TIME:
3731-     case DB_TYPE_TIMESTAMP:
3732-     case DB_TYPE_TIMESTAMPLTZ:
3733-     case DB_TYPE_TIMESTAMPTZ:
3734-     case DB_TYPE_DATE:
3735-     case DB_TYPE_DATETIME:

```

```cpp
3736-     case DB_TYPE_DATETIMELTZ:
3737-     case DB_TYPE_DATETIMETZ:
3738-     case DB_TYPE_MONETARY:
3739-     case DB_TYPE_SUB:
3740-     case DB_TYPE_POINTER:
3741-     case DB_TYPE_ERROR:
3742-     case DB_TYPE_SHORT:
3743-     case DB_TYPE_VOBJ:
3744-     case DB_TYPE_OID:
3745-     case DB_TYPE_NULL:
3746-     case DB_TYPE_VARIABLE:
3747-     case DB_TYPE_SET:
3748-     case DB_TYPE_MULTISET:
3749:     case DB_TYPE_SEQUENCE:
3750-     case DB_TYPE_DB_VALUE:
3751-     case DB_TYPE_VARCHAR:
3752-     case DB_TYPE_VARNCHAR:
3753-     case DB_TYPE_VARBIT:
3754-       found = d;
3755-       break;
3756-
3757-     case DB_TYPE_NUMERIC:
3758-       if (d->precision == domain->precision && d->scale == domain->scale)
3759-  {
3760-    found = d;
3761-  }
3762-       break;
3763-
3764-     case DB_TYPE_CHAR:
3765-     case DB_TYPE_NCHAR:
3766-     case DB_TYPE_BIT:
3767-       /* 1.deficient character related with CHAR & VARCHAR in set. ==> distinguishing VARCHAR from CHAR. 2.
3768-        * core dumped & deficient character related with CONST CHAR & CHAR in set. ==> In case of
3769-        * CHAR,NCHAR,BIT, cosidering precision. */



9022-    * This should still be an error, and the above
9023-    * code should have constructed a virtual mop.
9024-    * I'm not sure the rest of the code is consistent
9025-    * in this regard.
9026-    */
9027-        }
9028-      else
9029-        {
9030-   status = DOMAIN_INCOMPATIBLE;
9031-        }
9032-    }
9033-       }
9034-     db_make_object (target, v_obj);
9035-   }
9036-      }
9037-      break;
9038-#endif /* !SERVER_MODE */
9039-
9040-    case DB_TYPE_SET:
9041-    case DB_TYPE_MULTISET:
9042:    case DB_TYPE_SEQUENCE:
9043-      if (!TP_IS_SET_TYPE (original_type))
9044- {
9045-   status = DOMAIN_INCOMPATIBLE;
9046- }
9047-      else
9048- {
9049-   SETREF *setref;
9050-
9051-   setref = db_get_set (src);
9052-   if (setref)
9053-     {
9054-       TP_DOMAIN *set_domain;
9055-
9056-       set_domain = setobj_domain (setref->set);
9057-       if (src == dest && tp_domain_compatible (set_domain, desired_domain))
9058-  {
9059-    /*
9060-     * We know that this is a "coerce-in-place" operation, and
9061-     * we know that no coercion is necessary, so do nothing: we
9062-     * can use the exact same set without any conversion.



10458-      if (DB_IS_NULL (value2))
10459- {
10460-   result = (total_order ? DB_EQ : DB_UNK);
10461- }
10462-      else
10463- {
10464-   result = (total_order ? DB_LT : DB_UNK);
10465- }
10466-    }
10467-  else if (DB_IS_NULL (value2))
10468-    {
10469-      result = (total_order ? DB_GT : DB_UNK);
10470-    }
10471-  else
10472-    {
10473-      v1 = (DB_VALUE *) value1;
10474-      v2 = (DB_VALUE *) value2;
10475-
10476-      vtype1 = DB_VALUE_DOMAIN_TYPE (v1);
10477-      vtype2 = DB_VALUE_DOMAIN_TYPE (v2);
10478:      if (vtype1 != DB_TYPE_SET && vtype1 != DB_TYPE_MULTISET && vtype1 != DB_TYPE_SEQUENCE)
10479- {
10480-   return DB_NE;
10481- }
10482-
10483:      if (vtype2 != DB_TYPE_SET && vtype2 != DB_TYPE_MULTISET && vtype2 != DB_TYPE_SEQUENCE)
10484- {
10485-   return DB_NE;
10486- }
10487-
10488-      if (vtype1 != vtype2)
10489- {
10490-   if (!do_coercion)
10491-     {
10492-       /* types are not comparable */
10493-       return DB_NE;
10494-     }
10495-   else
10496-     {
10497-       db_make_null (&temp);
10498-       coercion = 1;
10499-       if (tp_more_general_type (vtype1, vtype2) > 0)
10500-  {
10501-    /* vtype1 is more general, coerce value 2 */
10502-    status = tp_value_coerce (v2, &temp, tp_domain_resolve_default (vtype1));
10503-    if (status != DOMAIN_COMPATIBLE)



11301-       fprintf (fp, ")");
11302-     }
11303-   break;
11304-
11305- case DB_TYPE_VARIABLE:
11306-   fprintf (fp, "union(");
11307-   fprint_domain (fp, d->setdomain);
11308-   fprintf (fp, ")");
11309-   break;
11310-
11311- case DB_TYPE_SET:
11312-   fprintf (fp, "set(");
11313-   fprint_domain (fp, d->setdomain);
11314-   fprintf (fp, ")");
11315-   break;
11316- case DB_TYPE_MULTISET:
11317-   fprintf (fp, "multiset(");
11318-   fprint_domain (fp, d->setdomain);
11319-   fprintf (fp, ")");
11320-   break;
11321: case DB_TYPE_SEQUENCE:
11322-   fprintf (fp, "sequence(");
11323-   fprint_domain (fp, d->setdomain);
11324-   fprintf (fp, ")");
11325-   break;
11326-
11327- case DB_TYPE_BIT:
11328- case DB_TYPE_VARBIT:
11329-   fprintf (fp, "%s(%d)", d->type->name, d->precision);
11330-   break;
11331-
11332- case DB_TYPE_CHAR:
11333- case DB_TYPE_VARCHAR:
11334-   fprintf (fp, "%s(%d) collate %s", d->type->name, d->precision, lang_get_collation_name (d->collation_id));
11335-   break;
11336-
11337- case DB_TYPE_NCHAR:
11338- case DB_TYPE_VARNCHAR:
11339-   fprintf (fp, "%s(%d) NATIONAL collate %s", d->type->name, d->precision,
11340-     lang_get_collation_name (d->collation_id));
11341-   break;



11433-}
11434-
11435-
11436-/*
11437- * tp_domain_references_objects - check if domain is an object domain or a
11438- * collection domain that might include objects.
11439- *    return: int (true or false)
11440- *    dom(in): the domain to be inspected
11441- */
11442-bool
11443-tp_domain_references_objects (const TP_DOMAIN * dom)
11444-{
11445-  switch (TP_DOMAIN_TYPE (dom))
11446-    {
11447-    case DB_TYPE_OBJECT:
11448-    case DB_TYPE_OID:
11449-    case DB_TYPE_VOBJ:
11450-      return true;
11451-    case DB_TYPE_SET:
11452-    case DB_TYPE_MULTISET:
11453:    case DB_TYPE_SEQUENCE:
11454-      dom = dom->setdomain;
11455-      if (dom)
11456- {
11457-   /*
11458-    * If domains are specified, we can assume that the upper levels
11459-    * have enforced the rule that no value in the collection has a
11460-    * domain that isn't included in this list.  If this list has no
11461-    * object domain, then no collection with this domain can include
11462-    * an object reference.
11463-    */
11464-   for (; dom; dom = dom->next)
11465-     {
11466-       if (tp_domain_references_objects (dom))
11467-  {
11468-    return true;
11469-  }
11470-     }
11471-
11472-   return false;
11473- }

```

```cpp
object_template.c
460-    {
461-      is_valid = 1;
462-    }
463-  else
464-    {
465-      /* wasn't on the first level cache, check the list */
466-      is_valid = ml_find (valid->validated_classes, class_);
467-      /* if its on the list, auto select this for the next time around */
468-      if (is_valid)
469-        {
470-   valid->last_class = class_;
471-        }
472-    }
473-       }
474-   }
475-      }
476-      break;
477-
478-    case DB_TYPE_SET:
479-    case DB_TYPE_MULTISET:
480:    case DB_TYPE_SEQUENCE:
481-      {
482- DB_SET *set;
483- DB_DOMAIN *domain;
484-
485- set = db_get_set (value);
486- domain = set_get_domain (set);
487- if (domain == valid->last_setdomain)
488-   {
489-     is_valid = 1;
490-   }
491-      }
492-      break;
493-
494-    case DB_TYPE_CHAR:
495-    case DB_TYPE_NCHAR:
496-    case DB_TYPE_VARCHAR:
497-    case DB_TYPE_VARNCHAR:
498-      if (type == valid->last_type && DB_GET_STRING_PRECISION (value) == valid->last_precision)
499- {
500-   is_valid = 1;



562- if (obj != NULL)
563-   {
564-     class_ = db_get_class (obj);
565-     if (class_ != NULL)
566-       {
567-  valid->last_class = class_;
568-  /*
569-   * !! note that we have to be building an external object list
570-   * here so these serve as GC roots.  This is kludgey, we should
571-   * be encapsulating structure rules inside cl_ where the
572-   * SM_VALIDATION is allocated.
573-   */
574-  (void) ml_ext_add (&valid->validated_classes, class_, NULL);
575-       }
576-   }
577-      }
578-      break;
579-
580-    case DB_TYPE_SET:
581-    case DB_TYPE_MULTISET:
582:    case DB_TYPE_SEQUENCE:
583-      {
584- DB_SET *set;
585-
586- set = db_get_set (value);
587- valid->last_setdomain = set_get_domain (set);
588-      }
589-      break;
590-
591-    case DB_TYPE_CHAR:
592-    case DB_TYPE_NCHAR:
593-    case DB_TYPE_VARCHAR:
594-    case DB_TYPE_VARNCHAR:
595-    case DB_TYPE_BIT:
596-    case DB_TYPE_VARBIT:
597-      valid->last_type = type;
598-      valid->last_precision = db_value_precision (value);
599-      valid->last_scale = 0;
600-      break;
601-
602-    case DB_TYPE_NUMERIC:

```

```cpp
class_object.c
1389-      assert (er_errid () != NO_ERROR);
1390-      return er_errid ();
1391-    }
1392-
1393-  pk_property = db_get_set (&prop_val);
1394-  err = set_get_element (pk_property, 1, &pk_val);
1395-  if (err != NO_ERROR)
1396-    {
1397-      goto end;
1398-    }
1399-
1400-  pk_seq = db_get_set (&pk_val);
1401-  size = set_size (pk_seq);
1402-
1403-  err = set_get_element (pk_seq, size - 3, &fk_container_val);
1404-  if (err != NO_ERROR)
1405-    {
1406-      goto end;
1407-    }
1408-
1409:  if (DB_VALUE_TYPE (&fk_container_val) == DB_TYPE_SEQUENCE)
1410-    {
1411-      fk_container = db_get_set (&fk_container_val);
1412-      fk_container_pos = set_size (fk_container);
1413-      pk_seq_pos = size - 3;
1414-    }
1415-  else
1416-    {
1417-      fk_container = set_create_sequence (1);
1418-      if (fk_container == NULL)
1419- {
1420-   goto end;
1421- }
1422-      db_make_sequence (&fk_container_val, fk_container);
1423-      fk_container_pos = 0;
1424-      pk_seq_pos = size - 2;
1425-    }
1426-
1427-  fk_seq = classobj_make_foreign_key_ref_seq (fk_info);
1428-  if (fk_seq == NULL)
1429-    {



1552-      err = (er_errid () != NO_ERROR) ? er_errid () : ER_FAILED;
1553-      return err;
1554-    }
1555-
1556-  pk_property = db_get_set (&prop_val);
1557-  err = set_get_element (pk_property, 1, &pk_val);
1558-  if (err != NO_ERROR)
1559-    {
1560-      goto end;
1561-    }
1562-
1563-  pk_seq = db_get_set (&pk_val);
1564-  size = set_size (pk_seq);
1565-
1566-  err = set_get_element (pk_seq, size - 2, &fk_container_val);
1567-  if (err != NO_ERROR)
1568-    {
1569-      goto end;
1570-    }
1571-
1572:  if (DB_VALUE_TYPE (&fk_container_val) == DB_TYPE_SEQUENCE)
1573-    {
1574-      fk_container = db_get_set (&fk_container_val);
1575-      fk_container_len = set_size (fk_container);
1576-      pk_seq_pos = size - 2;
1577-
1578-      /* find the position of the existing FK ref */
1579-      for (i = 0; i < fk_container_len; i++)
1580- {
1581-   PRIM_SET_NULL (&fk_val);
1582-   PRIM_SET_NULL (&name_val);
1583-
1584-   err = set_get_element (fk_container, i, &fk_val);
1585-   if (err != NO_ERROR)
1586-     {
1587-       goto end;
1588-     }
1589-
1590-   fk_seq = db_get_set (&fk_val);
1591-
1592-   /* A shallow copy for btid_val is enough. So, no need pr_clear_val(&btid_val). */



1720-      assert (er_errid () != NO_ERROR);
1721-      return er_errid ();
1722-    }
1723-
1724-  pk_property = db_get_set (&prop_val);
1725-  err = set_get_element (pk_property, 1, &pk_val);
1726-  if (err != NO_ERROR)
1727-    {
1728-      goto end;
1729-    }
1730-
1731-  pk_seq = db_get_set (&pk_val);
1732-  fk_container_pos = set_size (pk_seq) - 3;
1733-
1734-  err = set_get_element (pk_seq, fk_container_pos, &fk_container_val);
1735-  if (err != NO_ERROR)
1736-    {
1737-      goto end;
1738-    }
1739-
1740:  if (DB_VALUE_TYPE (&fk_container_val) != DB_TYPE_SEQUENCE)
1741-    {
1742-      err = ER_SM_INVALID_PROPERTY;
1743-      er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, err, 0);
1744-      goto end;
1745-    }
1746-
1747-  fk_container = db_get_set (&fk_container_val);
1748-  fk_count = set_size (fk_container);
1749-
1750-  for (i = 0; i < fk_count; i++)
1751-    {
1752-      err = set_get_element (fk_container, i, &fk_val);
1753-      if (err != NO_ERROR)
1754- {
1755-   goto end;
1756- }
1757-
1758-      fk_seq = db_get_set (&fk_val);
1759-
1760-      err = set_get_element (fk_seq, 1, &btid_val);



2453-  int error = NO_ERROR;
2454-  bool ok = true;
2455-
2456-  /* Make sure that the DB_VALUES are initialized */
2457-  db_make_null (&ids_val);
2458-  db_make_null (&name_val);
2459-
2460-  max = set_size (seq);
2461-  for (i = 0; i < max && error == NO_ERROR && ok; i += 2)
2462-    {
2463-      /* Get the constraint name */
2464-      error = set_get_element (seq, i, &name_val);
2465-      if (error != NO_ERROR)
2466- {
2467-   continue;
2468- }
2469-      /* get the constraint value sequence */
2470-      error = set_get_element (seq, i + 1, &ids_val);
2471-      if (error == NO_ERROR)
2472- {
2473:   if (DB_VALUE_TYPE (&ids_val) == DB_TYPE_SEQUENCE)
2474-     {
2475-       ids_seq = db_get_set (&ids_val);
2476-       ok = classobj_cache_constraint_entry (db_get_string (&name_val), ids_seq, class_, constraint_type);
2477-     }
2478-   pr_clear_value (&ids_val);
2479- }
2480-      pr_clear_value (&name_val);
2481-    }
2482-
2483-  if (error != NO_ERROR)
2484-    {
2485-      goto error;
2486-    }
2487-
2488-  return ok;
2489-
2490-  /* Error Processing */
2491-error:
2492-  pr_clear_value (&ids_val);
2493-  pr_clear_value (&name_val);



2519-    {
2520-      if (att->constraints)
2521- {
2522-   classobj_free_constraint (att->constraints);
2523-   att->constraints = NULL;
2524- }
2525-    }
2526-
2527-  /*
2528-   *  Extract the constraint property and process
2529-   */
2530-  if (class_->properties == NULL)
2531-    {
2532-      return ok;
2533-    }
2534-
2535-  for (i = 0; i < num_constraint_types && ok; i++)
2536-    {
2537-      if (classobj_get_prop (class_->properties, Constraint_properties[i], &un_value) > 0)
2538- {
2539:   if (DB_VALUE_TYPE (&un_value) == DB_TYPE_SEQUENCE)
2540-     {
2541-       un_seq = db_get_set (&un_value);
2542-       ok = classobj_cache_constraint_list (un_seq, class_, Constraint_types[i]);
2543-     }
2544-   pr_clear_value (&un_value);
2545- }
2546-    }
2547-
2548-  return ok;
2549-}
2550-
2551-/* SM_CLASS_CONSTRAINT */
2552-/*
2553- * classobj_find_attribute_list() - Alternative to classobj_find_attribute when we're
2554- *    looking for an attribute on a particular list.
2555- *   return: attribute structure
2556- *   attlist(in): list of attributes
2557- *   name(in): name to look for
2558- *   id(in): attribute id
2559- */



3031-      goto error;
3032-    }
3033-
3034-  buffer = db_get_string (&fvalue);
3035-  buffer_len = db_get_string_size (&fvalue);
3036-  filter_predicate->pred_stream = (char *) db_ws_alloc (buffer_len * sizeof (char));
3037-  if (filter_predicate->pred_stream == NULL)
3038-    {
3039-      goto error;
3040-    }
3041-
3042-  memcpy (filter_predicate->pred_stream, buffer, buffer_len);
3043-  filter_predicate->pred_stream_size = buffer_len;
3044-
3045-  if (set_get_element (pred_seq, 2, &avalue) != NO_ERROR)
3046-    {
3047-      er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_SM_INVALID_PROPERTY, 0);
3048-      goto error;
3049-    }
3050-
3051:  if (DB_VALUE_TYPE (&avalue) != DB_TYPE_SEQUENCE)
3052-    {
3053-      er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_SM_INVALID_PROPERTY, 0);
3054-      goto error;
3055-    }
3056-
3057-  att_seq = db_get_set (&avalue);
3058-  filter_predicate->num_attrs = att_seq_size = set_size (att_seq);
3059-  if (att_seq_size == 0)
3060-    {
3061-      filter_predicate->att_ids = NULL;
3062-    }
3063-  else
3064-    {
3065-      filter_predicate->att_ids = (int *) db_ws_alloc (sizeof (int) * att_seq_size);
3066-      if (filter_predicate->att_ids == NULL)
3067- {
3068-   goto error;
3069- }
3070-
3071-      for (i = 0; i < att_seq_size; i++)



3138-
3139-  /* make sure these are initialized for the error cleanup code */
3140-  db_make_null (&pvalue);
3141-  db_make_null (&uvalue);
3142-  db_make_null (&bvalue);
3143-  db_make_null (&avalue);
3144-  db_make_null (&fvalue);
3145-  db_make_null (&cvalue);
3146-  db_make_null (&statusval);
3147-
3148-  constraints = last = NULL;
3149-
3150-  /*
3151-   *  Process Index and Unique constraints
3152-   */
3153-  for (k = 0; k < num_constraint_types; k++)
3154-    {
3155-      if (classobj_get_prop (class_props, Constraint_properties[k], &pvalue) > 0)
3156- {
3157-   /* get the sequence & its size */
3158:   if (DB_VALUE_TYPE (&pvalue) != DB_TYPE_SEQUENCE)
3159-     {
3160-       goto structure_error;
3161-     }
3162-   props = db_get_set (&pvalue);
3163-   len = set_size (props);
3164-
3165-   /* this sequence is an alternating pair of constraint name & info sequence, as by: { name, { BTID,
3166-    * [att_name, asc_dsc], {fk_info | pk_info | prefix_length}, filter_predicate, status, comment }, name, { BTID,
3167-    * [att_name, asc_dsc], {fk_info | pk_info | prefix_length}, filter_predicate, status, comment }, ... } */
3168-   for (i = 0; i < len; i += 2)
3169-     {
3170-
3171-       /* get the name */
3172-       if (set_get_element (props, i, &uvalue))
3173-  {
3174-    goto other_error;
3175-  }
3176-       if (DB_VALUE_TYPE (&uvalue) != DB_TYPE_STRING)
3177-  {
3178-    goto structure_error;



3184-       if (new_ == NULL)
3185-  {
3186-    goto memory_error;
3187-  }
3188-       if (constraints == NULL)
3189-  {
3190-    constraints = new_;
3191-  }
3192-       else
3193-  {
3194-    last->next = new_;
3195-  }
3196-       last = new_;
3197-
3198-       /* Get the information sequence, this sequence contains a BTID followed by the names/ids of each
3199-        * attribute. */
3200-       if (set_get_element (props, i + 1, &uvalue))
3201-  {
3202-    goto structure_error;
3203-  }
3204:       if (DB_VALUE_TYPE (&uvalue) != DB_TYPE_SEQUENCE)
3205-  {
3206-    goto structure_error;
3207-  }
3208-
3209-       info = db_get_set (&uvalue);
3210-       info_len = set_size (info);
3211-
3212-       att_cnt = (info_len - 3) / 2; /* excludes BTID and comment */
3213-       assert (att_cnt > 0);
3214-
3215-       e = 0;
3216-
3217-       /* get the btid */
3218-       if (set_get_element (info, e++, &bvalue))
3219-  {
3220-    goto structure_error;
3221-  }
3222-       if (classobj_btid_from_property_value (&bvalue, &new_->index_btid, (char **) &new_->shared_cons_name))
3223-  {
3224-    goto structure_error;



3233-    goto memory_error;
3234-  }
3235-
3236-       new_->asc_desc = (int *) db_ws_alloc (sizeof (int) * att_cnt);
3237-       if (new_->asc_desc == NULL)
3238-  {
3239-    goto memory_error;
3240-  }
3241-       asc_desc = (int *) new_->asc_desc;
3242-
3243-       att = NULL;
3244-       /* Find each attribute referenced by the constraint. */
3245-       for (j = 0; j < att_cnt; j++)
3246-  {
3247-    /* name( or id) */
3248-    if (set_get_element (info, e++, &avalue))
3249-      {
3250-        goto structure_error;
3251-      }
3252-
3253:    if (DB_VALUE_TYPE (&avalue) == DB_TYPE_SEQUENCE)
3254-      {
3255-        new_->attributes[j] = NULL;
3256-        pr_clear_value (&avalue);
3257-        break;
3258-      }
3259-
3260-    if (DB_VALUE_TYPE (&avalue) == DB_TYPE_STRING)
3261-      {
3262-        att = classobj_find_attribute_list (attributes, db_get_string (&avalue), -1);
3263-      }
3264-    else if (DB_VALUE_TYPE (&avalue) == DB_TYPE_INTEGER)
3265-      {
3266-        att = classobj_find_attribute_list (attributes, NULL, db_get_int (&avalue));
3267-      }
3268-    else
3269-      {
3270-        goto structure_error;
3271-      }
3272-
3273-    new_->attributes[j] = att;



3324-      {
3325-        goto structure_error;
3326-      }
3327-    fk = db_get_set (&bvalue);
3328-
3329-    new_->fk_info = classobj_make_foreign_key_info (fk, new_->name, attributes);
3330-    if (new_->fk_info == NULL)
3331-      {
3332-        goto structure_error;
3333-      }
3334-
3335-    pr_clear_value (&bvalue);
3336-  }
3337-       else if (Constraint_types[k] == SM_CONSTRAINT_PRIMARY_KEY)
3338-  {
3339-    if (set_get_element (info, e++, &bvalue))
3340-      {
3341-        goto structure_error;
3342-      }
3343-
3344:    if (DB_VALUE_TYPE (&bvalue) == DB_TYPE_SEQUENCE)
3345-      {
3346-        new_->fk_info = classobj_make_foreign_key_ref_list (db_get_set (&bvalue));
3347-        if (new_->fk_info == NULL)
3348-   {
3349-     goto structure_error;
3350-   }
3351-
3352-        pr_clear_value (&bvalue);
3353-      }
3354-  }
3355-       else
3356-  {
3357-    if (set_get_element (info, e++, &bvalue))
3358-      {
3359-        goto structure_error;
3360-      }
3361-
3362:    if (DB_VALUE_TYPE (&bvalue) == DB_TYPE_SEQUENCE)
3363-      {
3364-        DB_SEQ *seq = db_get_set (&bvalue);
3365-        if (set_get_element (seq, 0, &fvalue))
3366-   {
3367-     pr_clear_value (&bvalue);
3368-     goto structure_error;
3369-   }
3370-        if (DB_VALUE_TYPE (&fvalue) == DB_TYPE_INTEGER)
3371-   {
3372-     new_->attrs_prefix_length = classobj_make_index_prefix_info (seq, att_cnt);
3373-     if (new_->attrs_prefix_length == NULL)
3374-       {
3375-         goto structure_error;
3376-       }
3377-   }
3378:        else if (DB_VALUE_TYPE (&fvalue) == DB_TYPE_SEQUENCE)
3379-   {
3380-     DB_SET *seq = db_get_set (&bvalue);
3381-     DB_SET *child_seq = db_get_set (&fvalue);
3382-     int seq_size = set_size (seq);
3383-     int flag;
3384-
3385-     j = 0;
3386-     while (true)
3387-       {
3388-         flag = 0;
3389-         if (set_get_element (child_seq, 0, &avalue) != NO_ERROR)
3390-    {
3391-      goto structure_error;
3392-    }
3393-
3394-         if (DB_IS_NULL (&avalue) || DB_VALUE_TYPE (&avalue) != DB_TYPE_STRING)
3395-    {
3396-      goto structure_error;
3397-    }
3398-
3399-         if (strcmp (db_get_string (&avalue), SM_FILTER_INDEX_ID) == 0)
3400-    {
3401-      flag = 0x01;
3402-    }
3403-         else if (strcmp (db_get_string (&avalue), SM_FUNCTION_INDEX_ID) == 0)
3404-    {
3405-      flag = 0x02;
3406-    }
3407-         else if (strcmp (db_get_string (&avalue), SM_PREFIX_INDEX_ID) == 0)
3408-    {
3409-      flag = 0x03;
3410-    }
3411-
3412-         pr_clear_value (&avalue);
3413-
3414-         if (set_get_element (child_seq, 1, &avalue) != NO_ERROR)
3415-    {
3416-      goto structure_error;
3417-    }
3418-
3419:         if (DB_VALUE_TYPE (&avalue) != DB_TYPE_SEQUENCE)
3420-    {
3421-      goto structure_error;
3422-    }
3423-
3424-         switch (flag)
3425-    {
3426-    case 0x01:
3427-      new_->filter_predicate = classobj_make_index_filter_pred_info (db_get_set (&avalue));
3428-      break;
3429-
3430-    case 0x02:
3431-      new_->func_index_info = classobj_make_function_index_info (db_get_set (&avalue));
3432-      break;
3433-
3434-    case 0x03:
3435-      new_->attrs_prefix_length =
3436-        classobj_make_index_prefix_info (db_get_set (&avalue), att_cnt);
3437-      break;
3438-
3439-    default:
3440-      break;
3441-    }
3442-
3443-         pr_clear_value (&avalue);
3444-
3445-         j++;
3446-         if (j >= seq_size)
3447-    {
3448-      break;
3449-    }
3450-
3451-         pr_clear_value (&fvalue);
3452-         if (set_get_element (seq, j, &fvalue) != NO_ERROR)
3453-    {
3454-      goto structure_error;
3455-    }
3456-
3457:         if (DB_VALUE_TYPE (&fvalue) != DB_TYPE_SEQUENCE)
3458-    {
3459-      goto structure_error;
3460-    }
3461-
3462-         child_seq = db_get_set (&fvalue);
3463-       }
3464-
3465-     if (new_->func_index_info)
3466-       {
3467-         /* function index and prefix length not allowed, yet */
3468-         new_->attrs_prefix_length = (int *) db_ws_alloc (sizeof (int) * att_cnt);
3469-         if (new_->attrs_prefix_length == NULL)
3470-    {
3471-      goto structure_error;
3472-    }
3473-         for (j = 0; j < att_cnt; j++)
3474-    {
3475-      new_->attrs_prefix_length[j] = -1;
3476-    }
3477-       }



8470-static int
8471-classobj_check_function_constraint_info (DB_SEQ * constraint_seq, bool * has_function_constraint)
8472-{
8473-  DB_VALUE bvalue, fvalue, avalue;
8474-  int constraint_seq_len = set_size (constraint_seq);
8475-  int j = 0;
8476-
8477-  assert (constraint_seq != NULL && has_function_constraint != NULL);
8478-
8479-  /* initializations */
8480-  *has_function_constraint = false;
8481-  db_make_null (&bvalue);
8482-  db_make_null (&avalue);
8483-  db_make_null (&fvalue);
8484-
8485-  if (set_get_element (constraint_seq, constraint_seq_len - 3, &bvalue) != NO_ERROR)
8486-    {
8487-      goto structure_error;
8488-    }
8489-
8490:  if (DB_VALUE_TYPE (&bvalue) == DB_TYPE_SEQUENCE)
8491-    {
8492-      DB_SEQ *seq = db_get_set (&bvalue);
8493-      if (set_get_element (seq, 0, &fvalue) != NO_ERROR)
8494- {
8495-   pr_clear_value (&bvalue);
8496-   goto structure_error;
8497- }
8498-      if (DB_VALUE_TYPE (&fvalue) == DB_TYPE_INTEGER)
8499- {
8500-   /* don't care about prefix length */
8501- }
8502:      else if (DB_VALUE_TYPE (&fvalue) == DB_TYPE_SEQUENCE)
8503- {
8504-   DB_SET *child_seq = db_get_set (&fvalue);
8505-   int seq_size = set_size (seq);
8506-
8507-   j = 0;
8508-   while (true)
8509-     {
8510-       if (set_get_element (child_seq, 0, &avalue) != NO_ERROR)
8511-  {
8512-    goto structure_error;
8513-  }
8514-
8515-       if (DB_IS_NULL (&avalue) || DB_VALUE_TYPE (&avalue) != DB_TYPE_STRING)
8516-  {
8517-    goto structure_error;
8518-  }
8519-
8520-       if (strcmp (db_get_string (&avalue), SM_FUNCTION_INDEX_ID) == 0)
8521-  {
8522-    *has_function_constraint = true;
8523-    pr_clear_value (&avalue);
8524-    break;
8525-  }
8526-
8527-       pr_clear_value (&avalue);
8528-
8529-       j++;
8530-       if (j >= seq_size)
8531-  {
8532-    break;
8533-  }
8534-
8535-       pr_clear_value (&fvalue);
8536-       if (set_get_element (seq, j, &fvalue) != NO_ERROR)
8537-  {
8538-    goto structure_error;
8539-  }
8540-
8541:       if (DB_VALUE_TYPE (&fvalue) != DB_TYPE_SEQUENCE)
8542-  {
8543-    goto structure_error;
8544-  }
8545-
8546-       child_seq = db_get_set (&fvalue);
8547-     }
8548- }
8549-      else
8550- {
8551-   goto structure_error;
8552- }
8553-
8554-      pr_clear_value (&fvalue);
8555-      pr_clear_value (&bvalue);
8556-    }
8557-  else
8558-    {
8559-      goto structure_error;
8560-    }
8561-

```

```cpp
object_domain.h
207-     * m. Only used in very special circumstances where we're trying to avoid copying
208-     * strings. */
209-
210-  TP_SET_MATCH
211-} TP_MATCH;
212-
213-/*
214- * TP_IS_SET_TYPE
215- *    Macros for detecting the set types, saves a function call.
216- */
217-/*
218- * !!! DB_TYPE_VOBJ probably should be added to this macro as this
219- * is now the behavior of pr_is_set_type() which we should try to
220- * phase out in favor of the faster inline macro.  Unfortunately, there
221- * are a number of usages of both TP_IS_SET_TYPE that may break if
222- * we change the semantics.  Will have to think carefully about this
223- */
224-
225-#define TP_IS_SET_TYPE(typenum) \
226-  ((((typenum) == DB_TYPE_SET) || ((typenum) == DB_TYPE_MULTISET) || \
227:    ((typenum) == DB_TYPE_SEQUENCE)) ? true : false)
228-
229-/*
230- * TP_IS_BIT_TYPE
231- *    Tests to see if the type id is one of the binary string types.
232- */
233-
234-#define TP_IS_BIT_TYPE(typeid) \
235-  (((typeid) == DB_TYPE_VARBIT) || ((typeid) == DB_TYPE_BIT))
236-
237-/*
238- * TP_IS_CHAR_TYPE
239- *    Tests to see if a type is any one of the character types.
240- */
241-
242-#define TP_IS_CHAR_TYPE(typeid) \
243-  (((typeid) == DB_TYPE_VARCHAR)  || ((typeid) == DB_TYPE_CHAR) || \
244-   ((typeid) == DB_TYPE_VARNCHAR) || ((typeid) == DB_TYPE_NCHAR))
245-
246-#define TP_IS_LOB_TYPE(typeid) \
247-  (((typeid) == DB_TYPE_BLOB)  || ((typeid) == DB_TYPE_CLOB))

```

```cpp
object_primitive.c
1599-  mr_setval_multiset,
1600-  mr_data_lengthmem_set,
1601-  mr_data_lengthval_set,
1602-  mr_data_writemem_set,
1603-  mr_data_readmem_set,
1604-  mr_data_writeval_set,
1605-  mr_data_readval_set,
1606-  NULL,    /* index_lengthmem */
1607-  NULL,    /* index_lengthval */
1608-  NULL,    /* index_writeval */
1609-  NULL,    /* index_readval */
1610-  NULL,    /* index_cmpdisk */
1611-  mr_freemem_set,
1612-  mr_data_cmpdisk_set,
1613-  mr_cmpval_set
1614-};
1615-
1616-PR_TYPE *tp_Type_multiset = &tp_Multiset;
1617-
1618-PR_TYPE tp_Sequence = {
1619:  "sequence", DB_TYPE_SEQUENCE, 1, sizeof (SETOBJ *), 0, 4,
1620-  mr_initmem_set,
1621-  mr_initval_sequence,
1622-  mr_setmem_set,
1623-  mr_getmem_sequence,
1624-  mr_setval_sequence,
1625-  mr_data_lengthmem_set,
1626-  mr_data_lengthval_set,
1627-  mr_data_writemem_set,
1628-  mr_data_readmem_set,
1629-  mr_data_writeval_set,
1630-  mr_data_readval_set,
1631-  NULL,    /* index_lengthmem */
1632-  NULL,    /* index_lengthval */
1633-  NULL,    /* index_writeval */
1634-  NULL,    /* index_readval */
1635-  NULL,    /* index_cmpdisk */
1636-  mr_freemem_set,
1637-  mr_data_cmpdisk_sequence,
1638-  mr_cmpval_sequence
1639-};



1963-     {
1964-       db_private_free (NULL, const_cast < char *>(value->data.json.schema_raw));
1965-       value->data.json.schema_raw = NULL;
1966-     }
1967- }
1968-      else
1969- {
1970-   value->data.json.document = NULL;
1971-   value->data.json.schema_raw = NULL;
1972- }
1973-      break;
1974-
1975-    case DB_TYPE_OBJECT:
1976-      /* we need to be sure to NULL the object pointer so that this db_value does not cause garbage collection problems
1977-       * by retaining an object pointer. */
1978-      value->data.op = NULL;
1979-      break;
1980-
1981-    case DB_TYPE_SET:
1982-    case DB_TYPE_MULTISET:
1983:    case DB_TYPE_SEQUENCE:
1984-    case DB_TYPE_VOBJ:
1985-      set_free (db_get_set (value));
1986-      value->data.set = NULL;
1987-      break;
1988-
1989-    case DB_TYPE_MIDXKEY:
1990-      midxkey_buf = value->data.midxkey.buf;
1991-      if (midxkey_buf != NULL)
1992- {
1993-   if (value->need_clear)
1994-     {
1995-       db_private_free_and_init (NULL, midxkey_buf);
1996-     }
1997-   /*
1998-    * Ack, phfffft!!! why should we have to know about the
1999-    * internals here?
2000-    */
2001-   value->data.midxkey.buf = NULL;
2002- }
2003-      break;



6937-  }
6938-     }
6939-   else
6940-     {
6941-       ref = set_copy (src_ref);
6942-       if (ref == NULL)
6943-  {
6944-    goto err_set;
6945-  }
6946-     }
6947- }
6948-
6949-      switch (set_type)
6950- {
6951- case DB_TYPE_SET:
6952-   db_make_set (dest, ref);
6953-   break;
6954- case DB_TYPE_MULTISET:
6955-   db_make_multiset (dest, ref);
6956-   break;
6957: case DB_TYPE_SEQUENCE:
6958-   db_make_sequence (dest, ref);
6959-   break;
6960- default:
6961-   break;
6962- }
6963-    }
6964-  else
6965-    {
6966-      db_value_domain_init (dest, set_type, DB_DEFAULT_PRECISION, DB_DEFAULT_SCALE);
6967-    }
6968-  return error;
6969-
6970-err_set:
6971-  /* couldn't allocate storage for set */
6972-  assert (er_errid () != NO_ERROR);
6973-  error = er_errid ();
6974-  switch (set_type)
6975-    {
6976-    case DB_TYPE_SET:
6977-      db_make_set (dest, NULL);
6978-      break;
6979-    case DB_TYPE_MULTISET:
6980-      db_make_multiset (dest, NULL);
6981-      break;
6982:    case DB_TYPE_SEQUENCE:
6983-      db_make_sequence (dest, NULL);
6984-      break;
6985-    default:
6986-      break;
6987-    }
6988-  db_make_null (dest);
6989-  return error;
6990-}
6991-
6992-static int
6993-mr_setval_set (DB_VALUE * dest, const DB_VALUE * src, bool copy)
6994-{
6995-  return mr_setval_set_internal (dest, src, copy, DB_TYPE_SET);
6996-}
6997-
6998-static int
6999-mr_data_lengthmem_set (void *memptr, TP_DOMAIN * domain, int disk)
7000-{
7001-  int size;
7002-



7240-       return ER_FAILED;
7241-     }
7242-   else
7243-     {
7244-       ref = setobj_get_reference (set);
7245-       if (ref == NULL)
7246-  {
7247-    or_abort (buf);
7248-    return ER_FAILED;
7249-  }
7250-       else
7251-  {
7252-    switch (set_get_type (ref))
7253-      {
7254-      case DB_TYPE_SET:
7255-        db_make_set (value, ref);
7256-        break;
7257-      case DB_TYPE_MULTISET:
7258-        db_make_multiset (value, ref);
7259-        break;
7260:      case DB_TYPE_SEQUENCE:
7261-        db_make_sequence (value, ref);
7262-        break;
7263-      default:
7264-        break;
7265-      }
7266-  }
7267-     }
7268- }
7269-      else
7270- {
7271-   /* copy == false, which means don't translate it into memory rep */
7272-   ref = set_make_reference ();
7273-   if (ref == NULL)
7274-     {
7275-       or_abort (buf);
7276-       return ER_FAILED;
7277-     }
7278-   else
7279-     {
7280-       int disk_size;



7301-    disk_size = or_disk_set_size (buf, domain, &set_type);
7302-  }
7303-
7304-       /* Record the pointer to the disk bits */
7305-       ref->disk_set = buf->ptr;
7306-       ref->need_clear = false;
7307-       ref->disk_size = disk_size;
7308-       ref->disk_domain = domain;
7309-
7310-       /* advance the buffer as if we had read the set */
7311-       rc = or_advance (buf, disk_size);
7312-
7313-       switch (set_type)
7314-  {
7315-  case DB_TYPE_SET:
7316-    db_make_set (value, ref);
7317-    break;
7318-  case DB_TYPE_MULTISET:
7319-    db_make_multiset (value, ref);
7320-    break;
7321:  case DB_TYPE_SEQUENCE:
7322-    db_make_sequence (value, ref);
7323-    break;
7324-  default:
7325-    break;
7326-  }
7327-     }
7328- }
7329-    }
7330-  return rc;
7331-}
7332-
7333-static void
7334-mr_freemem_set (void *memptr)
7335-{
7336-  /* since we aren't explicitly setting the set to NULL, we must set up the reference structures so they will get the
7337-   * new set when it is brought back in, this is the only primitive type for which the free function is semantically
7338-   * different than using the setmem function with a NULL value */
7339-
7340-  SETOBJ **mem = (SETOBJ **) memptr;
7341-



7417- }
7418-    }
7419-  /* NOTE: assumes that ownership info will already have been set or will be set by the caller */
7420-
7421-  return error;
7422-}
7423-
7424-static int
7425-mr_setval_multiset (DB_VALUE * dest, const DB_VALUE * src, bool copy)
7426-{
7427-  return mr_setval_set_internal (dest, src, copy, DB_TYPE_MULTISET);
7428-}
7429-
7430-/*
7431- * TYPE SEQUENCE
7432- */
7433-
7434-static void
7435-mr_initval_sequence (DB_VALUE * value, int precision, int scale)
7436-{
7437:  db_value_domain_init (value, DB_TYPE_SEQUENCE, precision, scale);
7438-  db_make_sequence (value, NULL);
7439-}
7440-
7441-static int
7442-mr_getmem_sequence (void *memptr, TP_DOMAIN * domain, DB_VALUE * value, bool copy)
7443-{
7444-  SETOBJ **mem = (SETOBJ **) memptr;
7445-  int error = NO_ERROR;
7446-  SETOBJ *set;
7447-  SETREF *ref;
7448-
7449-  set = *mem;
7450-  if (set == NULL)
7451-    {
7452-      error = db_make_sequence (value, NULL);
7453-    }
7454-  else
7455-    {
7456-      ref = setobj_get_reference (set);
7457-      if (ref)



7459-   error = db_make_sequence (value, ref);
7460- }
7461-      else
7462- {
7463-   assert (er_errid () != NO_ERROR);
7464-   error = er_errid ();
7465-   (void) db_make_sequence (value, NULL);
7466- }
7467-    }
7468-  /*
7469-   * NOTE: assumes that ownership info will already have been set or will
7470-   * be set by the caller
7471-   */
7472-
7473-  return error;
7474-}
7475-
7476-static int
7477-mr_setval_sequence (DB_VALUE * dest, const DB_VALUE * src, bool copy)
7478-{
7479:  return mr_setval_set_internal (dest, src, copy, DB_TYPE_SEQUENCE);
7480-}
7481-
7482-static DB_VALUE_COMPARE_RESULT
7483-mr_data_cmpdisk_sequence (void *mem1, void *mem2, TP_DOMAIN * domain, int do_coercion, int total_order, int *start_colp)
7484-{
7485-  DB_VALUE_COMPARE_RESULT c;
7486-  SETOBJ *seq1 = NULL, *seq2 = NULL;
7487-
7488-  assert (domain != NULL);
7489-
7490-  /* is not index type */
7491-  assert (!domain->is_desc && !tp_valid_indextype (TP_DOMAIN_TYPE (domain)));
7492-
7493-  mem1 = or_unpack_set ((char *) mem1, &seq1, domain);
7494-  mem2 = or_unpack_set ((char *) mem2, &seq2, domain);
7495-
7496-  if (seq1 == NULL || seq2 == NULL)
7497-    {
7498-      return DB_UNK;
7499-    }

```

```cpp
transform.c
221-  {NULL, (DB_TYPE) 0, 0, NULL, 0, 0, NULL}
222-};
223-META_CLASS tf_Metaclass_class = { META_CLASS_NAME, {META_PAGE_CLASS, 0, META_VOLUME}, 0, 0, 0,
224-&class_atts[0]
225-};
226-
227-/* QUERY_SPEC */
228-static META_ATTRIBUTE query_spec_atts[] = {
229-  {"specification", DB_TYPE_STRING, 1, NULL, 0, 0, NULL},
230-  {NULL, (DB_TYPE) 0, 0, NULL, 0, 0, NULL}
231-};
232-META_CLASS tf_Metaclass_query_spec = { META_QUERY_SPEC_NAME, {META_PAGE_QUERY_SPEC, 0, META_VOLUME}, 0, 0, 0,
233-&query_spec_atts[0]
234-};
235-
236-/* PARTITION */
237-static META_ATTRIBUTE partition_atts[] = {
238-  {"ptype", DB_TYPE_INTEGER, 1, NULL, 0, 0, NULL},
239-  {"pname", DB_TYPE_STRING, 1, NULL, 0, 0, NULL},
240-  {"pexpr", DB_TYPE_STRING, 1, NULL, 0, 0, NULL},
241:  {"pvalues", DB_TYPE_SEQUENCE, 0, NULL, 0, 0, NULL},
242-  {"comment", DB_TYPE_STRING, 1, NULL, 0, 0, NULL},
243-  {NULL, (DB_TYPE) 0, 0, NULL, 0, 0, NULL}
244-};
245-
246-META_CLASS tf_Metaclass_partition = { META_PARTITION_NAME, {META_PAGE_PARTITION, 0, META_VOLUME}, 0, 0, 0,
247-&partition_atts[0]
248-};
249-
250-/* ROOT */
251-static META_ATTRIBUTE root_atts[] = {
252-  {"heap_fileid", DB_TYPE_INTEGER, 0, NULL, 0, 0, NULL},
253-  {"heap_volid", DB_TYPE_INTEGER, 0, NULL, 0, 0, NULL},
254-  {"heap_pageid", DB_TYPE_INTEGER, 0, NULL, 0, 0, NULL},
255-  {"name", DB_TYPE_STRING, 0, NULL, 0, 0, NULL},
256-  {NULL, (DB_TYPE) 0, 0, NULL, 0, 0, NULL}
257-};
258-META_CLASS tf_Metaclass_root = { "rootclass", {META_PAGE_ROOT, 0, META_VOLUME}, 0, 0, 0, &root_atts[0] };
259-
260-/*
261- * Meta_classes



278-  &tf_Metaclass_partition,
279-  NULL
280-};
281-
282-#if !defined(CS_MODE)
283-
284-static CT_ATTR ct_class_atts[] = {
285-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
286-  {"inst_attr_count", NULL_ATTRID, DB_TYPE_INTEGER},
287-  {"shared_attr_count", NULL_ATTRID, DB_TYPE_INTEGER},
288-  {"inst_meth_count", NULL_ATTRID, DB_TYPE_INTEGER},
289-  {"class_meth_count", NULL_ATTRID, DB_TYPE_INTEGER},
290-  {"class_attr_count", NULL_ATTRID, DB_TYPE_INTEGER},
291-  {"is_system_class", NULL_ATTRID, DB_TYPE_INTEGER},
292-  {"class_type", NULL_ATTRID, DB_TYPE_INTEGER},
293-  {"owner", NULL_ATTRID, DB_TYPE_OBJECT},
294-  {"collation_id", NULL_ATTRID, DB_TYPE_INTEGER},
295-  {"tde_algorithm", NULL_ATTRID, DB_TYPE_INTEGER},
296-  {"unique_name", NULL_ATTRID, DB_TYPE_VARCHAR},
297-  {"class_name", NULL_ATTRID, DB_TYPE_VARCHAR},
298:  {"sub_classes", NULL_ATTRID, DB_TYPE_SEQUENCE},
299:  {"super_classes", NULL_ATTRID, DB_TYPE_SEQUENCE},
300:  {"inst_attrs", NULL_ATTRID, DB_TYPE_SEQUENCE},
301:  {"shared_attrs", NULL_ATTRID, DB_TYPE_SEQUENCE},
302:  {"class_attrs", NULL_ATTRID, DB_TYPE_SEQUENCE},
303:  {"inst_meths", NULL_ATTRID, DB_TYPE_SEQUENCE},
304:  {"class_meths", NULL_ATTRID, DB_TYPE_SEQUENCE},
305:  {"meth_files", NULL_ATTRID, DB_TYPE_SEQUENCE},
306:  {"query_specs", NULL_ATTRID, DB_TYPE_SEQUENCE},
307:  {"indexes", NULL_ATTRID, DB_TYPE_SEQUENCE},
308-  {"comment", NULL_ATTRID, DB_TYPE_VARCHAR},
309:  {"partition", NULL_ATTRID, DB_TYPE_SEQUENCE}
310-};
311-
312-static CT_ATTR ct_attribute_atts[] = {
313-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
314-  {"attr_name", NULL_ATTRID, DB_TYPE_VARCHAR},
315-  {"attr_type", NULL_ATTRID, DB_TYPE_INTEGER},
316-  {"from_attr_name", NULL_ATTRID, DB_TYPE_VARCHAR},
317-  {"data_type", NULL_ATTRID, DB_TYPE_INTEGER},
318-  {"def_order", NULL_ATTRID, DB_TYPE_INTEGER},
319-  {"from_class_of", NULL_ATTRID, DB_TYPE_OBJECT},
320-  {"is_nullable", NULL_ATTRID, DB_TYPE_INTEGER},
321-  {"default_value", NULL_ATTRID, DB_TYPE_VARCHAR},
322:  {"domains", NULL_ATTRID, DB_TYPE_SEQUENCE},
323-  {"comment", NULL_ATTRID, DB_TYPE_VARCHAR}
324-};
325-
326-static CT_ATTR ct_attrid_atts[] = {
327-  {"id", NULL_ATTRID, DB_TYPE_INTEGER},
328-  {"name", NULL_ATTRID, DB_TYPE_VARCHAR}
329-};
330-
331-static CT_ATTR ct_domain_atts[] = {
332-  {"object_of", NULL_ATTRID, DB_TYPE_OBJECT},
333-  {"data_type", NULL_ATTRID, DB_TYPE_INTEGER},
334-  {"prec", NULL_ATTRID, DB_TYPE_INTEGER},
335-  {"scale", NULL_ATTRID, DB_TYPE_INTEGER},
336-  {"code_set", NULL_ATTRID, DB_TYPE_INTEGER},
337-  {"collation_id", NULL_ATTRID, DB_TYPE_INTEGER},
338-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
339:  {"enumeration", NULL_ATTRID, DB_TYPE_SEQUENCE},
340:  {"set_domains", NULL_ATTRID, DB_TYPE_SEQUENCE},
341-  {"json_schema", NULL_ATTRID, DB_TYPE_STRING}
342-};
343-
344-static CT_ATTR ct_method_atts[] = {
345-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
346-  {"meth_name", NULL_ATTRID, DB_TYPE_VARCHAR},
347-  {"meth_type", NULL_ATTRID, DB_TYPE_INTEGER},
348-  {"from_meth_name", NULL_ATTRID, DB_TYPE_VARCHAR},
349-  {"from_class_of", NULL_ATTRID, DB_TYPE_OBJECT},
350:  {"signatures", NULL_ATTRID, DB_TYPE_SEQUENCE}
351-};
352-
353-static CT_ATTR ct_methsig_atts[] = {
354-  {"meth_of", NULL_ATTRID, DB_TYPE_OBJECT},
355-  {"arg_count", NULL_ATTRID, DB_TYPE_INTEGER},
356-  {"func_name", NULL_ATTRID, DB_TYPE_VARCHAR},
357:  {"return_value", NULL_ATTRID, DB_TYPE_SEQUENCE},
358:  {"arguments", NULL_ATTRID, DB_TYPE_SEQUENCE}
359-};
360-
361-static CT_ATTR ct_metharg_atts[] = {
362-  {"meth_sig_of", NULL_ATTRID, DB_TYPE_OBJECT},
363-  {"data_type", NULL_ATTRID, DB_TYPE_INTEGER},
364-  {"index_of", NULL_ATTRID, DB_TYPE_INTEGER},
365:  {"domains", NULL_ATTRID, DB_TYPE_SEQUENCE}
366-};
367-
368-static CT_ATTR ct_methfile_atts[] = {
369-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
370-  {"from_class_of", NULL_ATTRID, DB_TYPE_OBJECT},
371-  {"path_name", NULL_ATTRID, DB_TYPE_VARCHAR}
372-};
373-
374-static CT_ATTR ct_queryspec_atts[] = {
375-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
376-  {"spec", NULL_ATTRID, DB_TYPE_VARCHAR}
377-};
378-
379-static CT_ATTR ct_resolution_atts[] = {
380-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
381-  {"alias", NULL_ATTRID, DB_TYPE_VARCHAR},
382-  {"namespace", NULL_ATTRID, DB_TYPE_INTEGER},
383-  {"res_name", NULL_ATTRID, DB_TYPE_VARCHAR}
384-};
385-
386-static CT_ATTR ct_index_atts[] = {
387-  {"class_of", NULL_ATTRID, DB_TYPE_OBJECT},
388-  {"index_name", NULL_ATTRID, DB_TYPE_VARCHAR},
389-  {"is_unique", NULL_ATTRID, DB_TYPE_INTEGER},
390-  {"key_count", NULL_ATTRID, DB_TYPE_INTEGER},
391:  {"key_attrs", NULL_ATTRID, DB_TYPE_SEQUENCE},
392-  {"is_reverse", NULL_ATTRID, DB_TYPE_INTEGER},
393-  {"is_primary_key", NULL_ATTRID, DB_TYPE_INTEGER},
394-  {"is_foreign_key", NULL_ATTRID, DB_TYPE_INTEGER},
395-  {"filter_expression", NULL_ATTRID, DB_TYPE_VARCHAR},
396-  {"have_function", NULL_ATTRID, DB_TYPE_INTEGER},
397-  {"comment", NULL_ATTRID, DB_TYPE_VARCHAR},
398-  {"status", NULL_ATTRID, DB_TYPE_INTEGER}
399-};
400-
401-static CT_ATTR ct_indexkey_atts[] = {
402-  {"index_of", NULL_ATTRID, DB_TYPE_OBJECT},
403-  {"key_attr_name", NULL_ATTRID, DB_TYPE_VARCHAR},
404-  {"key_order", NULL_ATTRID, DB_TYPE_INTEGER},
405-  {"asc_desc", NULL_ATTRID, DB_TYPE_INTEGER},
406-  {"key_prefix_length", NULL_ATTRID, DB_TYPE_INTEGER},
407-  {"func", NULL_ATTRID, DB_TYPE_VARCHAR}
408-};
409-
410-static CT_ATTR ct_partition_atts[] = {
411-  {"index_of", NULL_ATTRID, DB_TYPE_OBJECT},
412-  {"ptype", NULL_ATTRID, DB_TYPE_INTEGER},
413-  {"pname", NULL_ATTRID, DB_TYPE_VARCHAR},
414-  {"pexpr", NULL_ATTRID, DB_TYPE_VARCHAR},
415:  {"pvalues", NULL_ATTRID, DB_TYPE_SEQUENCE},
416-  {"comment", NULL_ATTRID, DB_TYPE_VARCHAR}
417-};
418-
419-CT_CLASS ct_Class = {
420-  CT_CLASS_NAME,
421-  OID_INITIALIZER,
422-  (sizeof (ct_class_atts) / sizeof (ct_class_atts[0])),
423-  ct_class_atts
424-};
425-
426-CT_CLASS ct_Attribute = {
427-  CT_ATTRIBUTE_NAME,
428-  OID_INITIALIZER,
429-  (sizeof (ct_attribute_atts) / sizeof (ct_attribute_atts[0])),
430-  ct_attribute_atts
431-};
432-
433-CT_CLASS ct_Attrid = {
434-  NULL,
435-  OID_INITIALIZER,

```

```cpp
object_accessor.c
544-   error = er_errid ();
545- }
546-    }
547-
548-  if (error == NO_ERROR)
549-    {
550-      /* assign the value */
551-      if (mem != NULL)
552- {
553-   switch (TP_DOMAIN_TYPE (att->domain))
554-     {
555-     case DB_TYPE_SET:
556-     default:
557-       db_make_set (&val, new_set);
558-       break;
559-
560-     case DB_TYPE_MULTISET:
561-       db_make_multiset (&val, new_set);
562-       break;
563-
564:     case DB_TYPE_SEQUENCE:
565-       db_make_sequence (&val, new_set);
566-       break;
567-     }
568-
569-   error = att->domain->type->setmem (mem, att->domain, &val);
570-   db_value_put_null (&val);
571-
572-   if (error == NO_ERROR)
573-     {
574-       if (new_set != NULL && new_set != setref)
575-  {
576-    set_free (new_set);
577-  }
578-     }
579- }
580-      else
581- {
582-   /*
583-    * remove ownership information in the current set,
584-    * need to be able to free this !!!



589-       error = set_disconnect (current_set);
590-     }
591-
592-   if (error == NO_ERROR)
593-     {
594-
595-       /* set the new value */
596-       if (new_set != NULL)
597-  {
598-    switch (TP_DOMAIN_TYPE (att->domain))
599-      {
600-      case DB_TYPE_SET:
601-      default:
602-        db_make_set (&att->default_value.value, new_set);
603-        break;
604-
605-      case DB_TYPE_MULTISET:
606-        db_make_multiset (&att->default_value.value, new_set);
607-        break;
608-
609:      case DB_TYPE_SEQUENCE:
610-        db_make_sequence (&att->default_value.value, new_set);
611-        break;
612-      }
613-  }
614-       else
615-  {
616-    db_make_null (&att->default_value.value);
617-  }
618-
619-       if (new_set != NULL)
620-  {
621-    new_set->ref_count++;
622-  }
623-     }
624- }
625-    }
626-
627-  return error;
628-}
629-



1262-    }
1263-
1264-  /* convert NULL sets to DB_TYPE_NULL */
1265-  if (set == NULL)
1266-    {
1267-      db_make_null (dest);
1268-    }
1269-  else
1270-    {
1271-      switch (TP_DOMAIN_TYPE (att->domain))
1272- {
1273- case DB_TYPE_SET:
1274- default:
1275-   db_make_set (dest, set);
1276-   break;
1277-
1278- case DB_TYPE_MULTISET:
1279-   db_make_multiset (dest, set);
1280-   break;
1281-
1282: case DB_TYPE_SEQUENCE:
1283-   db_make_sequence (dest, set);
1284-   break;
1285- }
1286-    }
1287-
1288-  return NO_ERROR;
1289-}
1290-
1291-/*
1292- * obj_get_value -
1293- *    return: int
1294- *    op(in): class or instance pointer
1295- *    att(in): attribute descriptor
1296- *    mem(in): instance memory pointer (only for instance attribute)
1297- *    source(out): alternate source value (optional)
1298- *    dest(out): destionation value container
1299- *
1300- * Note:
1301- *    This is the basic generic function for accessing an attribute
1302- *    value.  It will call one of the specialized accessor functions above



1641-
1642-   temp_type = DB_VALUE_TYPE (&temp_value);
1643-   if (!TP_IS_SET_TYPE (temp_type))
1644-     {
1645-       ERROR0 (error, ER_OBJ_INVALID_SET_IN_PATH);
1646-     }
1647-   else
1648-     {
1649-       for (end = token; char_isdigit (*end) && *end != '\0'; end++)
1650-  ;
1651-
1652-       nextdelim = *end;
1653-       *end = '\0';
1654-       if (end == token)
1655-  {
1656-    ERROR0 (error, ER_OBJ_INVALID_INDEX_IN_PATH);
1657-  }
1658-       else
1659-  {
1660-    index = atoi (token);
1661:    if (temp_type == DB_TYPE_SEQUENCE)
1662-      {
1663-        error = db_seq_get (db_get_set (&temp_value), index, &temp_value);
1664-      }
1665-    else
1666-      {
1667-        error = db_set_get (db_get_set (&temp_value), index, &temp_value);
1668-      }
1669-
1670-    if (error == NO_ERROR)
1671-      {
1672-        for (++end; nextdelim != ']' && nextdelim != '\0'; nextdelim = *end++)
1673-   ;
1674-        if (nextdelim != '\0')
1675-   {
1676-     nextdelim = *end;
1677-   }
1678-      }
1679-  }
1680-     }
1681- }



3887-obj_find_object_by_cons_and_key (MOP classop, SM_CLASS_CONSTRAINT * cons, DB_VALUE * key, AU_FETCHMODE fetchmode)
3888-{
3889-  DB_TYPE value_type;
3890-  int error;
3891-  DB_VALUE key_element;
3892-  DB_COLLECTION *keyset;
3893-  SM_ATTRIBUTE **att;
3894-  MOP obj;
3895-  OID unique_oid;
3896-  int r, i;
3897-
3898-  value_type = DB_VALUE_TYPE (key);
3899-  att = cons->attributes;
3900-  if (att == NULL || att[0]->domain == NULL)
3901-    {
3902-      return NULL;
3903-    }
3904-
3905-  obj = NULL;
3906-  error = ER_OBJ_OBJECT_NOT_FOUND;
3907:  if (value_type != DB_TYPE_SEQUENCE)
3908-    {
3909-      /* 1 column */
3910-      if (tp_domain_select (att[0]->domain, key, 1, TP_ANY_MATCH) == NULL)
3911- {
3912-   error = ER_OBJ_INVALID_ARGUMENTS;
3913-   goto error_return;
3914- }
3915-
3916-      if (value_type == DB_TYPE_OBJECT)
3917- {
3918-   r = flush_temporary_OID (classop, key);
3919-   if (r == TEMPOID_FLUSH_NOT_SUPPORT)
3920-     {
3921-       error = ER_OBJ_INVALID_ARGUMENTS;
3922-       goto error_return;
3923-     }
3924-   else if (r == TEMPOID_FLUSH_FAIL)
3925-     {
3926-       return NULL;
3927-     }

```

```cpp
virtual_object.c
353- new_set = db_get_set (destination_value);
354- set_size = db_set_cardinality (set);
355- for (set_index = 0; set_index < set_size; ++set_index)
356-   {
357-     error = db_set_get (set, set_index, &set_value);
358-     if (error != NO_ERROR)
359-       {
360-  continue;
361-       }
362-
363-     error = vid_convert_object_attr_value (attribute_p, &set_value, &new_set_value, has_object);
364-     if (error == NO_ERROR)
365-       {
366-  error = db_set_add (new_set, &new_set_value);
367-  pr_clear_value (&new_set_value);
368-       }
369-   }
370-      }
371-      break;
372-
373:    case DB_TYPE_SEQUENCE:
374-      {
375- set = db_get_set (source_value);
376- new_set = db_get_set (destination_value);
377- set_size = db_seq_size (set);
378- for (set_index = 0; set_index < set_size; ++set_index)
379-   {
380-     error = db_seq_get (set, set_index, &set_value);
381-     if (error != NO_ERROR)
382-       {
383-  continue;
384-       }
385-     error = vid_convert_object_attr_value (attribute_p, &set_value, &new_set_value, has_object);
386-     if (error == NO_ERROR)
387-       {
388-  error = db_seq_put (new_set, set_index, &new_set_value);
389-  pr_clear_value (&new_set_value);
390-       }
391-   }
392-      }
393-      break;



1143-      error = db_seq_get (col, attribute_p->order, &val);
1144-
1145-      if (error < 0)
1146- break;
1147-
1148-      mem = inst + attribute_p->offset;
1149-      switch (DB_VALUE_TYPE (&val))
1150- {
1151- case DB_TYPE_VOBJ:
1152-   {
1153-     error = vid_vobj_to_object (&val, &vmop);
1154-     if (!(error < 0))
1155-       {
1156-  db_value_domain_init (&val, DB_TYPE_OBJECT, DB_DEFAULT_PRECISION, DB_DEFAULT_SCALE);
1157-  db_make_object (&val, vmop);
1158-       }
1159-   }
1160-   break;
1161- case DB_TYPE_SET:
1162- case DB_TYPE_MULTISET:
1163: case DB_TYPE_SEQUENCE:
1164-   {
1165-     error = set_convert_oids_to_objects (db_get_set (&val));
1166-   }
1167-   break;
1168- default:
1169-   break;
1170- }
1171-      if (error < 0)
1172- break;
1173-      obj_assign_value (mop, attribute_p, mem, &val);
1174-      pr_clear_value (&val);
1175-    }
1176-
1177-  if (error < 0)
1178-    {
1179-      db_ws_free (inst);
1180-      mop->object = NULL;
1181-      return error;
1182-    }
1183-  /*



1522-      if (!vclass)
1523- {
1524-   /*
1525-    * with no view, then the result is the object
1526-    * we just calculated.
1527-    */
1528-   *mop = obj;
1529- }
1530-      else
1531- {
1532-   DB_VALUE temp;
1533-   if (obj)
1534-     {
1535-       /* a view on an updatable class or proxy */
1536-       db_make_object (&temp, obj);
1537-       flags = VID_UPDATABLE;
1538-       *mop = ws_vmop (vclass, flags, &temp);
1539-     }
1540-   else
1541-     {
1542:       if (keys.domain.general_info.type == DB_TYPE_SEQUENCE)
1543-  {
1544-    /*
1545-     * The vclass refers to a non-updatable view result.
1546-     * look it up or install it in the workspace.
1547-     */
1548-    flags = 0;
1549-    *mop = ws_vmop (vclass, flags, &keys);
1550-
1551-    if (*mop)
1552-      {
1553-        error = vid_build_non_upd_object (*mop, &keys);
1554-      }
1555-  }
1556-     }
1557- }
1558-
1559-      if (!*mop)
1560- {
1561-   if (error == NO_ERROR)
1562-     {



1588-  *mop = NULL;
1589-  /* make sure we have a reasonable argument */
1590-  if (!value)
1591-    {
1592-      return ER_GENERIC_ERROR;
1593-    }
1594-
1595-  /* OIDs must be turned into objects */
1596-  switch (DB_VALUE_TYPE (value))
1597-    {
1598-    case DB_TYPE_OID:
1599-      oid = (OID *) db_get_oid (value);
1600-      if (oid != NULL && !OID_ISNULL (oid))
1601- {
1602-   *mop = ws_mop (oid, NULL);
1603- }
1604-      break;
1605-
1606-    case DB_TYPE_SET:
1607-    case DB_TYPE_MULTISET:
1608:    case DB_TYPE_SEQUENCE:
1609-      return set_convert_oids_to_objects (db_get_set (value));
1610-
1611-    default:
1612-      break;
1613-    }
1614-  return NO_ERROR;
1615-}
1616-
1617-/*
1618- * vid_object_to_vobj() -
1619- *    return: NO_ERROR if all OK, a negative ER code otherwise
1620- *    obj(in): a virtual object instance in the workspace
1621- *    vobj(out): a DB_TYPE_VOBJ db_value
1622- *
1623- * Note:
1624- *   requires: obj is a virtual mop in the workspace
1625- *   modifies: vobj, set/seq memory pool
1626- *   effects : builds the DB_TYPE_VOBJ db_value form of the given virtual mop
1627- */
1628-int

```

```cpp
object_representation.c
2990-     }
2991-   break;
2992-
2993- case DB_TYPE_OBJECT:
2994- case DB_TYPE_OID:
2995-   /*
2996-    * If the include_classoids argument was specified, set a flag in the
2997-    * disk representation indicating the presence of the class oids.
2998-    * This isn't necessary when the domain is used for value tagging
2999-    * since the class OID can always be be gotten from the instance oid.
3000-    */
3001-   if (include_classoids)
3002-     {
3003-       carrier |= OR_DOMAIN_CLASS_OID_FLAG;
3004-       has_oid = 1;
3005-     }
3006-   break;
3007-
3008- case DB_TYPE_SET:
3009- case DB_TYPE_MULTISET:
3010: case DB_TYPE_SEQUENCE:
3011- case DB_TYPE_TABLE:
3012- case DB_TYPE_MIDXKEY:
3013-   /*
3014-    * we need to recursively store the sub-domains following this one,
3015-    * since sets can have empty domains we need a flag to indicate this.
3016-    */
3017-   if (d->setdomain != NULL)
3018-     {
3019-       carrier |= OR_DOMAIN_SET_DOMAIN_FLAG;
3020-       has_subdomain = 1;
3021-     }
3022-
3023-   if (id == DB_TYPE_MIDXKEY)
3024-     {
3025-       assert (d->precision > 0 && d->precision == tp_domain_size (d->setdomain));
3026-       precision = d->precision;
3027-     }
3028-
3029-   break;
3030-



3286-      {
3287-        precision = DB_MAX_VARCHAR_PRECISION;
3288-      }
3289-    else if (type == DB_TYPE_VARNCHAR)
3290-      {
3291-        precision = DB_MAX_VARNCHAR_PRECISION;
3292-      }
3293-    else if (type == DB_TYPE_VARBIT)
3294-      {
3295-        precision = DB_MAX_VARBIT_PRECISION;
3296-      }
3297-  }
3298-       break;
3299-
3300-     case DB_TYPE_OBJECT:
3301-       has_classoid = carrier & OR_DOMAIN_CLASS_OID_FLAG;
3302-       break;
3303-
3304-     case DB_TYPE_SET:
3305-     case DB_TYPE_MULTISET:
3306:     case DB_TYPE_SEQUENCE:
3307-     case DB_TYPE_TABLE:
3308-     case DB_TYPE_MIDXKEY:
3309-       has_setdomain = carrier & OR_DOMAIN_SET_DOMAIN_FLAG;
3310-       if (type == DB_TYPE_MIDXKEY)
3311-  {
3312-    precision = (carrier & OR_DOMAIN_PRECISION_MASK) >> OR_DOMAIN_PRECISION_SHIFT;
3313-  }
3314-       break;
3315-
3316-     case DB_TYPE_ENUMERATION:
3317-       has_enum = carrier & OR_DOMAIN_ENUMERATION_FLAG;
3318-       has_collation = ((carrier & OR_DOMAIN_ENUM_COLL_FLAG) == OR_DOMAIN_ENUM_COLL_FLAG);
3319-       break;
3320-
3321-     case DB_TYPE_JSON:
3322-       has_schema = (carrier & OR_DOMAIN_SCHEMA_FLAG) != 0;
3323-       break;
3324-
3325-     default:
3326-       break;



3695-    rc = or_get_oid (buf, &class_oid);
3696-    if (rc != NO_ERROR)
3697-      {
3698-        goto error;
3699-      }
3700-#if !defined (SERVER_MODE)
3701-    /* swizzle the pointer if we're on the client */
3702-    class_mop = ws_mop (&class_oid, NULL);
3703-#endif /* !SERVER_MODE */
3704-  }
3705-       else
3706-  {
3707-    OID_SET_NULL (&class_oid);
3708-    class_mop = NULL;
3709-  }
3710-       dom = tp_domain_find_object (type, &class_oid, class_mop, is_desc);
3711-       break;
3712-
3713-     case DB_TYPE_SET:
3714-     case DB_TYPE_MULTISET:
3715:     case DB_TYPE_SEQUENCE:
3716-     case DB_TYPE_TABLE:
3717-     case DB_TYPE_MIDXKEY:
3718-       if (carrier & OR_DOMAIN_SET_DOMAIN_FLAG)
3719-  {
3720-    /* has setdomain */
3721-    setdomain = unpack_domain_2 (buf, NULL);
3722-    if (setdomain == NULL)
3723-      {
3724-        goto error;
3725-      }
3726-  }
3727-       else
3728-  {
3729-    goto error;
3730-  }
3731-
3732-       if (type == DB_TYPE_MIDXKEY)
3733-  {
3734-    precision = (carrier & OR_DOMAIN_PRECISION_MASK) >> OR_DOMAIN_PRECISION_SHIFT;
3735-  }



5951-  return size;
5952-}
5953-
5954-/*
5955- * or_put_enumeration () - pack an enumeration
5956- *    return: error code or NO_ERROR
5957- *    enumeration (in): enumeration
5958- */
5959-int
5960-or_put_enumeration (OR_BUF * buf, const DB_ENUMERATION * enumeration)
5961-{
5962-  int rc = NO_ERROR, idx;
5963-  DB_VALUE value;
5964-  DB_ENUM_ELEMENT *db_enum = NULL;
5965-
5966-  if (enumeration->count == 0)
5967-    {
5968-      return rc;
5969-    }
5970-  /* an enumeration is packed as a collection of strings */
5971:  rc = or_put_set_header (buf, DB_TYPE_SEQUENCE, enumeration->count, 0, 0, 0, 0, 0);
5972-  if (rc != NO_ERROR)
5973-    {
5974-      return rc;
5975-    }
5976-
5977-  for (idx = 0; idx < enumeration->count; idx++)
5978-    {
5979-      db_enum = &enumeration->elements[idx];
5980-      db_make_varchar (&value, TP_FLOATING_PRECISION_VALUE, DB_GET_ENUM_ELEM_STRING (db_enum),
5981-         DB_GET_ENUM_ELEM_STRING_SIZE (db_enum), DB_GET_ENUM_ELEM_CODESET (db_enum),
5982-         enumeration->collation_id);
5983-      rc = tp_String.data_writeval (buf, &value);
5984-      pr_clear_value (&value);
5985-
5986-      if (rc != NO_ERROR)
5987- {
5988-   break;
5989- }
5990-    }
5991-

```

```cpp
set_object.c
840-       * sets & multisets, must set the size back down as the elements do not logically exist yet */
841-      err = col_expand (col, size - 1);
842-      if (err)
843- {
844-   setobj_free (col);
845-   return NULL;
846- }
847-      col->size = 0;
848-
849-      /* initialize the domain with one of the built in domain structures */
850-      if (col)
851- {
852-   switch (settype)
853-     {
854-     case DB_TYPE_SET:
855-       col->domain = &tp_Set_domain;
856-       break;
857-     case DB_TYPE_MULTISET:
858-       col->domain = &tp_Multiset_domain;
859-       break;
860:     case DB_TYPE_SEQUENCE:
861-       col->domain = &tp_Sequence_domain;
862-       break;
863-     case DB_TYPE_VOBJ:
864-       col->domain = &tp_Vobj_domain;
865-       break;
866-     default:
867-       er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_SET_INVALID_DOMAIN, 1, pr_type_name ((DB_TYPE) settype));
868-       setobj_free (col);
869-       col = NULL;
870-       break;
871-     }
872- }
873-    }
874-  return col;
875-}
876-
877-/*
878- * non_null_index() - search for the greatest index between a lower and
879- *                    upper bound which has a non NULL db_value.
880- *      return: long



1147-   /* ANSI puts NULLs at end of set collections for comparison */
1148-   /*
1149-    * since NULL is not equal to NULL, the insertion index
1150-    * returned for sequences might as well be at the end where
1151-    * its checp to insert.
1152-    */
1153-   insertindex = col->size;
1154- }
1155-      else
1156- {
1157-   /*
1158-    * hack, if the collection contains temporary OIDs, we're forced to
1159-    * use a linear search as the numeric OID can change without warning.
1160-    * Unfortunate but not very easy to perform effeciently without segmenting
1161-    * the collection into sorted and unsorted regions.
1162-    * Once the set is assured to contain only permanent OID's, the sort
1163-    * can be performed reliably.
1164-    *
1165-    */
1166-#if !defined(SERVER_MODE)
1167:   if (col->sorted && col->coltype != DB_TYPE_SEQUENCE && DB_VALUE_TYPE (val) == DB_TYPE_OBJECT)
1168-     {
1169-
1170-       DB_OBJECT *obj = db_get_object (val);
1171-       if (obj != NULL && OBJECT_HAS_TEMP_OID (obj))
1172-  {
1173-    /* we're inserting a temp OID, must force the collection to become unsorted */
1174-    col->sorted = 0;
1175-  }
1176-     }
1177-#endif
1178-
1179-   /*
1180-    * Unsorted sets were introduced to deal with temporary/permanent
1181-    * oids on the client, but are now also used as an optimization for
1182-    * multisets, which are now sorted only on demand. Sorting them
1183-    * on demand yields logarithmic instead of quadratic behavior.
1184-    *
1185-    */
1186-
1187:   if (col->coltype != DB_TYPE_SEQUENCE && col->sorted)
1188-     {
1189-       /* Start point for ordered search */
1190-       insertindex = non_null_index (col, 0, col->lastinsert);
1191-       if (insertindex < 0)
1192-  {
1193-    /*
1194-     * all set values were NULL. Will not find val.
1195-     * insert at beginning.
1196-     */
1197-    insertindex = 0;
1198-  }
1199-       else
1200-  {
1201-    /* determine which side of last insertion index to search from. */
1202-    compare = tp_value_compare (val, INDEX (col, insertindex), do_coerce, 1);
1203-    if (compare == DB_UNK)
1204-      {
1205-        insertindex = ER_GENERIC_ERROR;
1206-      }
1207-    else if (compare > 0)



1292-
1293-  if (!col || colindex < 0 || !val)
1294-    {
1295-      /* invalid args */
1296-      error = ER_GENERIC_ERROR;
1297-      return error;
1298-    }
1299-
1300-  error = col_expand (col, colindex);
1301-
1302-  if (!(error < 0))
1303-    {
1304-      if (!DB_IS_NULL (val))
1305- {
1306-   col->lastinsert = colindex;
1307- }
1308-      blockindex = BLOCK (colindex);
1309-      offset = OFFSET (colindex);
1310-
1311-      /* check for temporary OIDs, isn't this where we should be clearing the sorted flag too ? */
1312:      if (col->coltype != DB_TYPE_SEQUENCE && DB_VALUE_TYPE (val) == DB_TYPE_OBJECT)
1313- {
1314-#if defined (SERVER_MODE)
1315-   assert_release (false);
1316-   return ER_FAILED;
1317-#else /* !defined (SERVER_MODE) */
1318-   DB_OBJECT *obj = db_get_object (val);
1319-   if (obj != NULL && OBJECT_HAS_TEMP_OID (obj))
1320-     {
1321-       col->may_have_temporary_oids = 1;
1322-     }
1323-#endif /* !defined (SERVER_MODE) */
1324- }
1325-
1326-      /* If this should be cloned, the caller should do it. This primitive just allows the assignment to the right
1327-       * location in the collection */
1328-      col->array[blockindex][offset] = *val;
1329-    }
1330-
1331-  return error;
1332-}



1432-     {
1433-       memmove (&col->array[topblock][1], &col->array[topblock][0], topblockcount * sizeof (DB_VALUE));
1434-       col->array[topblock][0] = col->array[topblock - 1][BLOCKING_LESS1];
1435-       topblock--;
1436-       topblockcount = BLOCKING_LESS1;
1437-     }
1438-   topoffset = BLOCKING_LESS1;
1439- }
1440-      else
1441- {
1442-   topoffset = OFFSET (col->size - 1);
1443- }
1444-      /* shift the values from this block up one. */
1445-      while (topoffset > offset)
1446- {
1447-   col->array[blockindex][topoffset] = col->array[blockindex][topoffset - 1];
1448-   topoffset--;
1449- }
1450-
1451-      /* check for temporary OIDs, isn't this where we should be clearing the sorted flag too ? */
1452:      if (col->coltype != DB_TYPE_SEQUENCE && DB_VALUE_TYPE (val) == DB_TYPE_OBJECT)
1453- {
1454-#if defined (SERVER_MODE)
1455-   assert_release (false);
1456-   return ER_FAILED;
1457-#else /* !defined (SERVER_MODE) */
1458-   DB_OBJECT *obj = db_get_object (val);
1459-   if (obj != NULL && OBJECT_HAS_TEMP_OID (obj))
1460-     {
1461-       col->may_have_temporary_oids = 1;
1462-     }
1463-#endif /* !defined (SERVER_MODE)s */
1464- }
1465-
1466-      /* If this should be cloned, the caller should do it. This primitive just allows the assignment to the right
1467-       * location in the collection */
1468-      col->array[blockindex][offset] = *val;
1469-      PRIM_SET_NULL (val);
1470-      col->lastinsert = colindex;
1471-    }
1472-



1647- {
1648-   error = i;
1649-   return error;
1650- }
1651-      /* a MULTISET- insert it whether found or not */
1652-      if (found)
1653- {
1654-   i++;   /* insert at next index after last found */
1655- }
1656-
1657-      if (i < col->size - COL_BLOCK_SIZE)
1658- {
1659-   /* heuristic to avoid quadratic multiset creation. if we are moving more than a block, its likely we are
1660-    * entering quadratic behavior. Simply insert at the end, and mark the multiset unsorted. It will be sorted
1661-    * later on demand if need be, and this sort will be logarithmic instead of quadratic. */
1662-   i = col->size;
1663-   col->sorted = 0;
1664- }
1665-      error = col_insert (col, i, val);
1666-      break;
1667:    case DB_TYPE_SEQUENCE:
1668-    case DB_TYPE_VOBJ:
1669-      error = col_put (col, col->size, val);
1670-      break;
1671-    default:
1672-      /* bad args */
1673-      error = ER_GENERIC_ERROR;
1674-      break;
1675-    }
1676-  return error;
1677-}
1678-
1679-/*
1680- * col_drop() - drop val to col
1681- *      return: int
1682- *  col(in) :
1683- *  val(in) :
1684- *
1685- *  Note :
1686- *      if its a set - use set difference.
1687- *      if its a multiset - use multiset difference.



1692-col_drop (COL * col, DB_VALUE * val)
1693-{
1694-  int error = NO_ERROR;
1695-  long i, found, do_coerce;
1696-
1697-  if (!col || !val)
1698-    {
1699-      error = ER_GENERIC_ERROR; /* bad args */
1700-      return error;
1701-    }
1702-
1703-  do_coerce = (col->coltype == DB_TYPE_SET);
1704-  i = col_find (col, &found, val, do_coerce);
1705-  if (i < 0)
1706-    {
1707-      error = i;
1708-      return error;
1709-    }
1710-  if (found)
1711-    {
1712:      if (col->coltype == DB_TYPE_SEQUENCE)
1713- {
1714-   PRIM_SET_NULL (INDEX (col, i));
1715- }
1716-      else
1717- {
1718-   error = col_delete (col, i);
1719- }
1720-    }
1721-  return error;
1722-}
1723-
1724-/*
1725- * col_drop_nulls() - drop all nulls calues in col
1726- *      return: int
1727- *  col(in) :
1728- *
1729- */
1730-
1731-int
1732-col_drop_nulls (COL * col)



1860- *  set2(in) :
1861- *  do_coercion(in) :
1862- *  total_order(in) :
1863- *
1864- *  Note :
1865- *      We don't compare the vclass, but the proxyclass and
1866- *      the keys are compared.
1867- *
1868- *      There must be a total order on VOBJS since they can be used in merge
1869- *      joins as the join column and he can't deal with DB_UNK.  To accomplish
1870- *      this total order, we do a lexigraphical sort on proxyclass and keys.
1871- *
1872- *      We will return DB_UNK if the VOBJ is malformed.
1873- */
1874-
1875-DB_VALUE_COMPARE_RESULT
1876-setvobj_compare (COL * set1, COL * set2, int do_coercion, int total_order)
1877-{
1878-  DB_VALUE_COMPARE_RESULT cmp = DB_UNK;
1879-
1880:  if ((set1->size == 3) && (set2->size == 3) && (set1->coltype == DB_TYPE_SEQUENCE || set1->coltype == DB_TYPE_VOBJ)
1881:      && (set2->coltype == DB_TYPE_SEQUENCE || set2->coltype == DB_TYPE_VOBJ))
1882-    {
1883-      cmp = DB_EQ;
1884-      if (DB_VALUE_DOMAIN_TYPE (&set1->array[0][2]) != DB_TYPE_OID)
1885- {
1886-   cmp = tp_value_compare (&set1->array[0][1], &set2->array[0][1], do_coercion, 1);
1887- }
1888-      if (cmp == DB_EQ)
1889- {
1890-   cmp = tp_value_compare (&set1->array[0][2], &set2->array[0][2], do_coercion, total_order);
1891- }
1892-      else
1893- {
1894-   if (cmp == DB_UNK)
1895-     {
1896-       /* one of set1->array[0][1] or set2->array[0][2] must be NULL. */
1897-       cmp = DB_IS_NULL (&set1->array[0][1]) ? DB_LT : DB_GT;
1898-     }
1899- }
1900-    }
1901-  else if (debug_level > 0)



2416- *      return: DB_COLLECTION *
2417- *
2418- */
2419-
2420-DB_COLLECTION *
2421-set_create_multi (void)
2422-{
2423-  return set_create (DB_TYPE_MULTISET, 1);
2424-}
2425-
2426-/*
2427- * set_create_sequence()
2428- *      return: DB_COLLECTION *
2429- *  size(in) :
2430- *
2431- */
2432-
2433-DB_COLLECTION *
2434-set_create_sequence (int size)
2435-{
2436:  return set_create (DB_TYPE_SEQUENCE, size);
2437-}
2438-
2439-DB_COLLECTION *
2440-set_create_with_domain (TP_DOMAIN * domain, int initial_size)
2441-{
2442-  DB_COLLECTION *col;
2443-  COL *setobj;
2444-
2445-  col = NULL;
2446-  setobj = setobj_create_with_domain (domain, initial_size);
2447-  if (setobj == NULL)
2448-    {
2449-      return (col);
2450-    }
2451-
2452-  col = set_make_reference ();
2453-
2454-  if (col == NULL)
2455-    {
2456-      setobj_free (setobj);



4603- *    or an access failure on the owning object.
4604- */
4605-
4606-TP_DOMAIN *
4607-setobj_get_domain (COL * set)
4608-{
4609-  if (set->domain != NULL)
4610-    {
4611-      return set->domain;
4612-    }
4613-  /* Set without an owner and without a domain, this isn't supposed to happen any more.  Slam in one of the built-in
4614-   * domains. */
4615-  switch (set->coltype)
4616-    {
4617-    case DB_TYPE_SET:
4618-      set->domain = &tp_Set_domain;
4619-      break;
4620-    case DB_TYPE_MULTISET:
4621-      set->domain = &tp_Multiset_domain;
4622-      break;
4623:    case DB_TYPE_SEQUENCE:
4624-      set->domain = &tp_Sequence_domain;
4625-      break;
4626-    case DB_TYPE_VOBJ:
4627-      set->domain = &tp_Vobj_domain;
4628-      break;
4629-    default:
4630-      /* what is it? must be a structure error */
4631-      er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_SET_INVALID_DOMAIN, 1, "NULL set domain");
4632-      break;
4633-    }
4634-
4635-  return set->domain;
4636-}
4637-
4638-/*
4639- * swizzle_value() - This converts a value containing an OID into one
4640- *                   containing a DB_OBJECT*
4641- *      return: none
4642- *  val(in) : value to convert
4643- *  input(in) :



4886-  for (i = col->size - 1; i >= 0; i--)
4887-    {
4888-      var = INDEX (col, i);
4889-      if (filter)
4890- {
4891-   swizzle_value (var, 0);
4892- }
4893-      if (filter && DB_VALUE_TYPE (var) == DB_TYPE_OBJECT)
4894- {
4895-   error = check_set_object (var, &removed);
4896-   if (error < 0)
4897-     {
4898-       break;
4899-     }
4900-   if (!removed)
4901-     {
4902-       card++;
4903-     }
4904-   else
4905-     {
4906:       if (col->coltype != DB_TYPE_SEQUENCE)
4907-  {
4908-    col_delete (col, i);
4909-  }
4910-     }
4911- }
4912-      else if (!db_value_is_null (var))
4913- {
4914-   card++;
4915- }
4916-    }
4917-  if (cardptr == NULL)
4918-    {
4919-      return error;
4920-    }
4921-
4922-  if (error < 0)
4923-    {
4924-      *cardptr = 0;
4925-    }
4926-  else



5380- *      than appending.
5381- */
5382-
5383-int
5384-setobj_union (COL * set1, COL * set2, COL * result)
5385-{
5386-  int rc;
5387-  int error = NO_ERROR;
5388-  int index1, index2;
5389-  DB_VALUE *val1, *val2;
5390-
5391-  if (!set1->sorted)
5392-    {
5393-      error = setobj_sort (set1);
5394-    }
5395-  if (!error && !set2->sorted)
5396-    {
5397-      error = setobj_sort (set2);
5398-    }
5399-
5400:  if (result->coltype == DB_TYPE_SEQUENCE)
5401-    {
5402-      /* append sequences */
5403-      for (index1 = 0; index1 < set1->size && !(error < NO_ERROR); index1++)
5404- {
5405-   val1 = INDEX (set1, index1);
5406-   error = setobj_add_element (result, val1);
5407- }
5408-      for (index2 = 0; index2 < set2->size && !(error < NO_ERROR); index2++)
5409- {
5410-   val2 = INDEX (set2, index2);
5411-   error = setobj_add_element (result, val2);
5412- }
5413-    }
5414-  else
5415-    {
5416-      /* compare elements in ascending order */
5417-      index1 = 0;
5418-      index2 = 0;
5419-      while (index1 < set1->size && index2 < set2->size && !(error < NO_ERROR))
5420- {



5665-  for (i = 0; i < col->size && !(error < NO_ERROR); i++)
5666-    {
5667-      var = INDEX (col, i);
5668-      typ = DB_VALUE_DOMAIN_TYPE (var);
5669-      switch (typ)
5670- {
5671- case DB_TYPE_OID:
5672-   swizzle_value (var, 0);
5673-   break;
5674- case DB_TYPE_VOBJ:
5675-   error = vid_vobj_to_object (var, &mop);
5676-   if (!(error < 0))
5677-     {
5678-       pr_clear_value (var);
5679-       db_value_domain_init (var, DB_TYPE_OBJECT, DB_DEFAULT_PRECISION, DB_DEFAULT_SCALE);
5680-       db_make_object (var, mop);
5681-     }
5682-   break;
5683- case DB_TYPE_SET:
5684- case DB_TYPE_MULTISET:
5685: case DB_TYPE_SEQUENCE:
5686-   error = set_convert_oids_to_objects (db_get_set (var));
5687-   break;
5688- default:
5689-   break;
5690- }
5691-    }
5692-#endif
5693-
5694-  return (error);
5695-}
5696-
5697-/* SET ELEMENT ACCESS FUNCTIONS */
5698-
5699-/*
5700- * setobj_get_element_ptr() - This is used in controlled conditions to
5701- *                            get a direct pointer to a set element value.
5702- *      return: error code
5703- *  col(in) : collection object
5704- *  index(in) : element index
5705- *  result(in) : pointer to pointer to value (returned)

```

```cpp
authenticate_grant.cpp
832-  DB_VALUE value;
833-  DB_SET *grants = NULL;
834-  MOP grantor, gowner, class_;
835-  int gsize, i, j, existing, cache;
836-  bool need_pop_er_stack = false;
837-
838-  assert (grant_ptr != NULL);
839-
840-  *grant_ptr = NULL;
841-
842-  er_stack_push ();
843-
844-  need_pop_er_stack = true;
845-
846-  error = obj_get (auth, "grants", &value);
847-  if (error != NO_ERROR)
848-    {
849-      goto end;
850-    }
851-
852:  if (DB_VALUE_TYPE (&value) != DB_TYPE_SEQUENCE || DB_IS_NULL (&value) || db_get_set (&value) == NULL)
853-    {
854-      error = ER_AU_CORRUPTED;
855-      er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, error, 0);
856-
857-      goto end;
858-    }
859-
860-  if (!filter)
861-    {
862-      goto end;
863-    }
864-
865-  grants = db_get_set (&value);
866-  gsize = set_size (grants);
867-
868-  /* there might be errors */
869-  error = er_errid ();
870-  if (error != NO_ERROR)
871-    {
872-      goto end;

```

```cpp
object_printer.cpp
282- case DB_TYPE_JSON:
283-   strcpy (temp_buffer, temp_domain->type->name);
284-   ustr_upper (temp_buffer);
285-   if (temp_domain->json_validator != NULL)
286-     {
287-       m_buf ("%s(\'%s\')", temp_buffer, db_json_get_schema_raw_from_validator (temp_domain->json_validator));
288-     }
289-   else
290-     {
291-       m_buf (temp_buffer);
292-     }
293-   break;
294-
295- case DB_TYPE_NUMERIC:
296-   strcpy (temp_buffer, temp_domain->type->name);
297-   m_buf ("%s(%d,%d)", ustr_upper (temp_buffer), temp_domain->precision, temp_domain->scale);
298-   break;
299-
300- case DB_TYPE_SET:
301- case DB_TYPE_MULTISET:
302: case DB_TYPE_SEQUENCE:
303-   strcpy (temp_buffer, temp_domain->type->name);
304-   ustr_upper (temp_buffer);
305-   m_buf ("%s OF ", temp_buffer);
306-   if (temp_domain->setdomain != NULL)
307-     {
308-       if (temp_domain->setdomain->next != NULL && prt_type == class_description::SHOW_CREATE_TABLE)
309-  {
310-    m_buf ("(");
311-    describe_domain (*temp_domain->setdomain, prt_type, force_print_collation);
312-    m_buf (")");
313-  }
314-       else
315-  {
316-    describe_domain (*temp_domain->setdomain, prt_type, force_print_collation);
317-  }
318-     }
319-   break;
320-
321- case DB_TYPE_ENUMERATION:
322-   has_collation = 1;
```
