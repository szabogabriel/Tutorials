# Intro

The following examples are meant to copy-paste as template for `javax.persistence.criteria.CriteriaBuilder`.

```Java
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<MyEntity> cq = cb.createQuery(MyEntity.class);

    Root<MyEntity> root = cq.from(MyEntity.class);

    List<Predicate> predicatesList = new ArrayList<>();
    predicatesList.add(cb.equal(root.get(MyEntity_.FIELD_1), field1FilterValue));
    predicatesList.add(cb.equal(root.get(MyEntity_.FIELD_2), field2FilterValue));
    predicatesList.add(cb.equal(root.get(MyEntity_.Field_3), Boolean.FALSE));
    
    Predicate[] predicates = predicatesList.toArray(new Predicate[predicatesList.size()]);
    
    Order[] orders = new Order[1];
    orders[0] = cb.desc(root.get(My_Entity_.FIELD_0));

    cq.select(root).where(predicates).orderBy(orders);
    return entityManager.createQuery(cq).getResultList();
```
