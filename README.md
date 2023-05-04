# laminas-servicemanager

> Removed political appeal incompatible with free software here. The original repository is at this [link](https://github.com/laminas/laminas-servicemanager).

The Service Locator design pattern is implemented by the `Laminas\ServiceManager`
component. The Service Locator is a service/object locator, tasked with
retrieving other objects.

- File issues at https://github.com/laminas/laminas-servicemanager/issues
- [Online documentation](https://docs.laminas.dev/laminas-servicemanager)
- [Documentation source files](docs/book/)

## Benchmarks

We provide scripts for benchmarking laminas-servicemanager using the
[PHPBench](https://github.com/phpbench/phpbench) framework; these can be
found in the `benchmarks/` directory.

To execute the benchmarks you can run the following command:

```bash
$ vendor/bin/phpbench run --report=aggregate
```
