#### Опишите deployment для веб-приложения в kubernetes в виде yaml-манифеста. Оставляйте в коде комментарии по принятым решениям. Есть следующие вводные:

- у нас мультизональный кластер (три зоны), в котором пять нод
- приложение требует около 5-10 секунд для инициализации
- по результатам нагрузочного теста известно, что 4 пода справляются с пиковой нагрузкой
- на первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда “ровно” в районе 128M memory
- приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём
- хотим максимально отказоустойчивый deployment
- хотим минимального потребления ресурсов от этого deployment’а

____
Старался всё максимально расписать в манифестах, что бы было понятно проверяющему, соответственно там и "максимально отказоустойчевый deployment" и HPA 
и другие таски которые вы написали в задание.  
____
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1           
  selector:
    matchLabels:
      app: php-apache
  strategy:
    rollingUpdate:       # Используем стратегию rollingUpdate, то есть при обновлении приложения,для клиента будет происходить более прозрачно.
      maxSurge: 1        # На какое количество реплик можно увеличить, относительно "replicas: 1", при обновлении.
      maxUnavailable: 0  # На какое количество реплик можно понизить, относительно "replicas: 1", при обновлении, в данном случае получается на 0, так как реплика одна.
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - image: k8s.gcr.io/hpa-example
        name: php-apache
        ports:
        - containerPort: 80
        readinessProbe:            # Проверят , готово ли приложения принимать трафик. Если проба сфейлиться, приложение убирается из балансировки
          failureThreshold: 3      # Три попытки для выполнения readinessProbe
          httpGet:                 # Проверяет по http запросу, в корневой каталог, по 80 порту
            path: /
            port: 80
          periodSeconds: 10       # Проверяем каждый 10 секунд
          successThreshold: 1     # Количество сбойных проверок
          timeoutSeconds: 1       # timeout на выполнение нашей пробы
        livenessProbe:            # Проверяет состояния приложения во время его жизни ( если проба сфейлится - перезагрузит под)
          failureThreshold: 3
          httpGet:                # Проверяет по http запросу, в корневой каталог, по 80 порту
            path: /
            port: 80
          periodSeconds: 10       # Проверяем каждый 10 секунд
          successThreshold: 1     # Количество сбойных проверок
          timeoutSeconds: 1
          initialDelaySeconds: 14 # После того как приложение запустилось, ждём 14 секунд, прежде чем выполнять livenessProbe.
          # Так же можно использовать startupProme.
      # startupProbe: - соответственно после отработки startupProbe, запускаются остальные пробы.
      #   httpGet:
      #    path: /
      #    port: 80
      #   periodSeconds: 7
      #   failureThreshold: 2
        resources:
          requests:
            cpu: 100m
            memory: 128Mi     # Что касается ОЗУ, поточнее бы узнать потребление, а не в "районе" , что бы приложение не убилось.
          limits:
            cpu: 200m         # Максимальное потребление ЦПУ пода
            memory: 128Mi     # Максимальное потребление ОЗУ пода
```


Ну и соответственно нам нужна абстракция - service, для приём трафика.
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - port: 80            # На каком порту сервис принимает запрос
    targetPort: 80      # Порт в контейнере, куда сервис будет форвардить трафик.
  selector:
    app: php-apache     
  type: ClusterIP
  ```



Захотелось выбрать HPA v2,так как у него расшириные возможности,хотя в данной ситуации нам это не к чему, но хозяин - барин :)
Соответсвенно скейлим по CPU - 70% использования ресурса,относительно requestов на pode.
```
apiVersion: autoscaling/v2beta2 
kind: HorizontalPodAutoscaler    
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1         # Минимальное количество подов при скейлинге
  maxReplicas: 4         # Максимальное количество подов при скейлинге ( Количество взял из ТЗ)
metrics:
- type: Resource         # Выбрал тип ресурс, и скейлить будем по ЦПУ      
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
 ```




