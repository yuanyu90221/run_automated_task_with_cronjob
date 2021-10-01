# run_automated_task_with_cronjob

今天要來實作 [Run automated tasks with cron jobs](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/) 這個 Task

## Prequest

已有 k8s 執行個體

在這個章節將會使用 minikube

## 佈署目標

佈署一個每分鐘送一個 Hello 訊息到 Console 的任務到 k8s cluster 上

## 建立 CronJob

設定 cronjob.yaml

```yaml=
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

建立一個 CronJob

設定名稱為 hello

設定排程周期為 "*/1 * * * *"   每分鐘執行一次的意思, 語法如下

```yaml=
#      ┌────────────────── timezone (optional)
#      |      ┌───────────── minute (0 - 59)
#      |      │ ┌───────────── hour (0 - 23)
#      |      │ │ ┌───────────── day of the month (1 - 31)
#      |      │ │ │ ┌───────────── month (1 - 12)
#      |      │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
#      |      │ │ │ │ │                                   7 is also Sunday on some systems)
#      |      │ │ │ │ │
#      |      │ │ │ │ │
# CRON_TZ=UTC * * * * *
```
在 container 部份的設定是使用 busybox 的 image (有linux shell 的工具)

執行內容為使用 /bin/sh 印出現在日期以及 Hello from the Kubernetes cluster 

**特別的是** CronJob apiVersion 的值在 1.21.* 版之後就可以使用 batch/v1 

因為我這邊使用的 minikube 是使用 1.20.0 版本所以只能用 batch/v1beta1

建立佈署指令
```shell=
kubectl apply -f cronjob.yaml
```
![](https://i.imgur.com/abQmIOZ.png)


查詢佈署指令

```shell=
kubectl get cronjob hello
```

![](https://i.imgur.com/LbHwKsS.png)


## 驗證

找出 job 名稱

```shell=
kubectl get jobs --watch
```
![](https://i.imgur.com/dNfzttN.png)

每分鐘會產生一個 Job

透過 job name 找到 Pod 驗證內容

```shell=
pods=$(kubectl get pods --selector=job-name=hello-1633100400 --output=jsonpath={.items[*].metadata.name})
kubectl logs $pods
```
![](https://i.imgur.com/u9Q4wFY.png)


## 刪除 CronJob

```shell=
kubectl delete cronjob hello
```

## 後記

有幾個特別的可以設定在 Job tempate

### startingDeadline(Optional)

這個欄位代表要啟動 CronJob 的最晚時間

假設超過這個時間 CronJob 沒有執行, 之後就不會執行

如果沒有設定代表 CronJob 沒有 Deadline

### currencyPolicy(Optional)

這個欄位是用來指定是否能夠同時執行多個由 CronJob 產生的 Job

設定值如下：

**Allow(預設值)** 可以同時運行

**Forbid** 不允許多個 Job 同時運行

**Replace** 假設在新的 Job 要執行時, 舊的 Job 還沒執行結束, 則 CronJob 會用新的 Job 取代目前正在執行的 Job

### successfulJobsHistoryLimit

這個欄位是用來指定多少最新完成的 Job 能夠被保留

預設值是 3, 代表預設只保留最新 3 個完成的

### failedJobsHistoryLimit

這個欄位是用來指定多少最新失敗的 Job 能夠被保留

預設值是 1, 代表預設只保留最新 1 個失敗的