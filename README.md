# Horizontal Pod Autoscaler (HPA)

## Langkah 1: Aktifkan minikube dengan setting sbb:

```bash
$ minikube start \
	--extra-config=controller-manager.horizontal-pod-autoscaler-upscale-delay=1m \
	--extra-config=controller-manager.horizontal-pod-autoscaler-downscale-delay=1m \
	--extra-config=controller-manager.horizontal-pod-autoscaler-sync-period=10s \
	--extra-config=controller-manager.horizontal-pod-autoscaler-downscale-stabilization=1m
```

> Catatan:
>
> - Parameter diatas supaya mempercepat proses autoscale dan downscale dibanding dengan nilai defaultnya

## Langkah 2: Aktifkan metrics-server addons

```bash
$ minikube addons list
$ minikube addons enable metrics-server
```

## Langkah 3: Deploy php untuk membuat beban server

```yaml
$ vi php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 200m
            memory: 0.5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache 
```

## Langkah 4: Apply deployment

```bash
$ kubect create -f php-apache.yml
$ kubectl get deploy
$ kubectl get po
```

## Langkah 5: membuat HPA

```bash
$ kubectl autoscale deployment php-apache \
	--cpu-percent=50 
	--min=1 \
	--max=10
$ kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          53m
```

## Langkah 6: Tambahkan Beban

```bash
$ kubectl run -it load-generator \
	--rm \
	--image=busybox \
	--restart=Never -- \
	/bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

- setelah beberapa menit (di terminal yg lain):

```bash
$ watch kubectl get hpa
```

> Catatan:
>
> - Harusnya Target% akan bertambah sesuai dengan load yang 
> - Setelah kenaikan target% otomatis akan menambah replicas

- di terminal lain

```bash
$ kubectl get deployment php-apache
```



## Langkah 7: Menghentikan Load

- Untuk menghentikan load gunakan ctrl-c pada terminal yang menjalankan load-generator

```bash
$ kubectl get hpa
$ kubectl get deployment php-apache
```

> Catatan:
>
> - Perhatikan target% akan turun dan otomatis mengurangi replica sampai dengan 1.