---
title: JDK-JUC-CompletableFuture
date: 2020-10-17 19:52:16
categories: [java,juc,oom]
tags:
---

接口批量查询

```java
 public List<HcmMobileIMDataDto> queryIMUserInfo(Long entId, Long buId, List<Integer> types, List<Long> employeeIds) {
       return ...;
    }

    public List<HcmMobileIMDataDto> queryIMUserInfoPartitionAsync(Long entId, Long buId, List<Integer> types, List<Long> employeeIds) {
        final ExecutorService executorService = Executors.newFixedThreadPool(5);
        if (CollectionUtils.isEmpty(employeeIds)) {
            return new ArrayList<>(0);
        }
        //并行
        List<HcmMobileIMDataDto> resultList = partitionCall2ListAsync(employeeIds, EMPLOYEE_SIZE_LIMIT, executorService, (eachList) -> queryIMUserInfo(entId,buId,types, eachList));
        return resultList;
    }



    private <T, V> List<V> partitionCall2ListAsync(List<T> dataList,
                                                         int size,
                                                         ExecutorService executorService,
                                                         Function<List<T>, List<V>> function) {

        List<CompletableFuture<List<V>>> completableFutures = Lists.partition(dataList, size)
                .stream()
                .map(eachList -> {
                    return CompletableFuture.supplyAsync(() -> function.apply(eachList), executorService);
                })
                .collect(Collectors.toList());


        CompletableFuture<Void> allFinished = CompletableFuture.allOf(completableFutures.toArray(new CompletableFuture[0]));
        log.info("partitionCall2ListAsync执行，等待批量结果ing...");
        try {
            allFinished.get();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        log.info("partitionCall2ListAsync执行，等待结束");
        List<V> ans =completableFutures.stream()
                .map(CompletableFuture::join)
                .filter(e->e!=null)
                .reduce(new ArrayList<V>(), ((list1, list2) -> {
                    List<V> resultList = new ArrayList<>();
                        resultList.addAll(list1);
                        resultList.addAll(list2);
                    return resultList;
                }));
        log.info("partitionCall2ListAsync执行完成, ans={}",ans);
        return ans;
    }
```

