## Example create helm chart

### local server start

```bash
$ helm repo list
NAME  	URL
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts

$ helm serve &
Regenerating index. This may take a moment.

Now serving you on 127.0.0.1:8879
```

### create chart template

```bash
$ helm create echo
Creating echo

$ tree .
echo/
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

### packaging

```bash
$ helm package echo
Successfully packaged chart and saved it to: /PATH/TO/echo-0.1.0.tgz

$ helm search echo
NAME      	CHART VERSION	APP VERSION	DESCRIPTION
local/echo	0.1.0        	1.0        	A Helm chart for Kubernetes
```

### install

```bash
$ helm install -f echo.yaml --name echo local/echo
```