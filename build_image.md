## 构建镜像的最佳实践

### 1 ENTRYPOINT和CMD

ENTRYPOINT用于指定容器启动的命令，CMD用于指定容器启动时的默认参数

### 2 shell和exec

`ENTRYPOINT node app.js`：在shell进程中启动node，容器中的初始进程是shell

`ENTRYPOINT ["node", "app.js"]`：直接在容器中启动node，容器中的初始进程是node
