# 加速 ci 流水线

1. 缓存相关依赖
   1. 使用本地依赖
   2. 使用 docker build 依赖
   3. 使用 buildkite 插件进行挂载
2. 精简 build/package tasks
3. 提升测试速度
   1. 按照测试金字塔安排合理的测试数量和比例
   2. 重用 application context
4. 缩小部署镜像体积：减少 pod 拉镜像时间
   1. docker file multi-stage build
5. 合理应用并行步骤

相关阅读：[gradle build 从 5 分钟到 1 分钟](gradle-build-optimization.md)
