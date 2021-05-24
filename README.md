# Helm Chart for ClickHouse

[ClickHouse](https://clickhouse.tech/) is an open source column-oriented database management system capable of real time generation of analytical data reports using 
SQL queries.
This repository provides a helm chart for easily setting up a ClickHouse cluster in Kubernetes.

The development of this helm was sponsored and supported by [extrument.com](https://extrument.com/).


## Installation

### Add Helm Repository

```
helm repo add radonck https://radondb.github.io/clickhouse-cluster-helm/
helm repo update
```

### Install to Kubernetes

```
helm install clickhouse radonck/clickhouse -f values.yaml
```

For a list of all configurable options and variables see [values.yaml](values.yaml).


## License

This helm chart is published under the Apache License, Version 2.0.
See [LICENSE.md](LICENSE.md) for more information.

Copyright (c) by [TCeason](https://tceason.github.io/) &lt;[tai_chong@foxmail.com](tai_chong@foxmail.com)&gt; .


### Attributions

  * **ClickHouse**
    * Project URL: https://clickhouse.tech/
    * License: Apache License, Version 2.0

