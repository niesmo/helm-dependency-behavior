# Helm Dependency Behavior Report
This repo is meant to display some weird behavior that I was experincing when using helm.

## Setup
The setup that I have here is a 2 chart setup.

### `random-template`
`random-template` chart is a simple helm chart with the following template:
```yaml
global:
  name: {{ .Values.global.name }}
  theActualCar: {{ .Values.global.carModel }}
  globaldata:
    - /straight/from/template
  {{- range $key, $value := .Values.global.globalData }}
    - {{ $key }}
  {{- end }}
data:
  - /straight/from/template
{{- range $key, $value := .Values.data }}
  - {{ $key }}
{{- end }}
```
The template also includes the following `values.yaml` so that we can directly template it out.
```yaml
global:
  name: 🤡
  carModel: 🚗
  globalData:
    /random/template:
data:
  /random/template:
```

### `main-chart`
This chart depends on the `random-template` chart using a file reference.

This chart include a `values.yaml` with the following content
```yaml
global:
  name: 🤡
  globalData:
    /parent/chart:
data:
  /parent/chart:
```

## What's the issue?
The issue is related to how helm templates behave when used directly vs when they are set as the dependent in another chart.

when I run `helm template .`, what I expect here is
```yaml
# Source: main-chart/charts/random-template/templates/random.yaml
global:
  name: 🤡
  theActualCar: 🚗
  globaldata:
    - /straight/from/template
    - /random/template
    - /parent/chart
data:
  - /straight/from/template
  - /random/template
  - /parent/chart
```
But this is what I get
```yaml
# Source: main-chart/charts/random-template/templates/random.yaml
global:
  name: 🤡
  theActualCar: 🚗
  globaldata:
    - /straight/from/template
    - /random/template
data:
  - /straight/from/template
```
Questions:
1. Why does `/parent/chart` not show up under `globaldata`
1. Why do either `/random/template` or `/parent/chart` not show up under `data`?


Now if I run `helm template charts/random-template`, I get exactly what I expect which is
```yaml
# Source: random-template/templates/random.yaml
global:
  name: 🤡
  theActualCar: 🚗
  globaldata:
    - /straight/from/template
    - /random/template
data:
  - /straight/from/template
  - /random/template
```

Now if I add any value to the `.global.globaldata."/parent/chart"` it results in it showing up in the result of `helm template .`

```yaml
global:
  name: 🤡
  globalData:
    /parent/chart: meaningless_value
data:
  /parent/chart:
```
results in
```yaml
# Source: main-chart/charts/random-template/templates/random.yaml
global:
  name: 🤡
  theActualCar: 🚗
  globaldata:
    - /straight/from/template
    - /parent/chart
    - /random/template
data:
  - /straight/from/template
```

HOWEVER, if I add the same meaningless_value to the `.data./parent/chart"`, it does not have the same effect 😭.
```yaml
global:
  name: 🤡
  globalData:
    /parent/chart: meaningless_value
data:
  /parent/chart: meaningless_value
```
results in
```yaml
# Source: main-chart/charts/random-template/templates/random.yaml
global:
  name: 🤡
  theActualCar: 🚗
  globaldata:
    - /straight/from/template
    - /parent/chart
    - /random/template
data:
  - /straight/from/template
```

Questions:
1. Why does having a value in the map impact the way the `range` function renders the list?
1. Why does the behavior inconsistent between `.global.globaldata` and `.data`?