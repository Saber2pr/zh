报错信息 pip install pmdarima: ModuleNotFoundError: No module named 'numpy'

```sh
ERROR: Failed to build installable wheels for some pyproject.toml based projects (pmdarima)
pip install pmdarima:
  ModuleNotFoundError: No module named 'numpy'
```

freqtrade 镜像版本降低到 2025.5 及之前版本即可，Dockerfile 示例：

```dockerfile
FROM freqtradeorg/freqtrade:2025.5

RUN pip install pmdarima
```

### 宝贝推广

【量化策略37套附6套正向回测结果】 https://m.tb.cn/h.hqpATDP?tk=TRtH471WgWF