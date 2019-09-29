# thread

## 1. 同步

## 2. 通信

### 2.1 wait, notify, notifyAll

1. 必须在 sync {} 内
2. notify先发生, 却没有 wait, 会导致 notify 信号丢失
3. interrupt 是修改状态, 并不会丢失终端状态
4. sync 会先将**工作内存**数据清空, 重新拉去
5. 