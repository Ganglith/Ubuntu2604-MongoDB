# Ubuntu 26.04 安装 MongoDB 实录：内核 7.0 下如何让 8.0 正常启动

> 环境：Ubuntu 26.04.1 LTS (resolute) / Kernel 7.0.0-38-generic / x86_64
> 结论先说：**装 8.0.20，并锁死版本**。9.0.x 和 8.0.23+ 在 kernel ≥ 6.19 上会被 mongod 主动拒绝启动。

---

## 一、问题现象

按官方文档在 Ubuntu 26.04 上装最新版 MongoDB，安装过程一切正常，但服务起不来：

```bash
sudo systemctl status mongod
# Active: failed (Result: exit-code)
```

日志里的关键报错：

```json
{"t":{"$date":"2026-10-08T20:27:26.566+08:00"},"s":"F","c":"CONTROL","id":12257600,
 "ctx":"main","msg":"MongoDB cannot start: Linux kernel versions 6.19 and newer has a known
 incompatibility with this version of MongoDB. See https://jira.mongodb.org/browse/SERVER-121912
 for more information."}
```

查看日志的命令：

```bash
sudo journalctl -u mongod --no-pager -n 50
```

**注意：这是「能装不能跑」**——`apt install` 成功不代表服务能启动。

---

## 二、根因定位

- Linux 内核 **6.19** 调整了 `rseq`（restartable sequences）相关机制，MongoDB 底层使用的内存分配器（TCMalloc）依赖旧行为，在新内核上运行会触发已知崩溃风险。
- MongoDB 的处置方式：**在启动阶段做内核版本检查，kernel ≥ 6.19 直接拒绝启动**，宁可不让跑也不冒数据风险。
- 该检查是**后续补丁版本才加入**的：8.0.23+ 和 9.0.x 都有，**8.0.20 没有**。

官方 issue：<https://jira.mongodb.org/browse/SERVER-121912>

### 实测版本对照（Ubuntu 26.04 + Kernel 7.0.0）

| MongoDB 版本 | 安装 | 启动 | 说明 |
|-------------|:----:|:----:|------|
| 9.0.2 | ✅ | ❌ | 内核检查拦截 |
| 8.0.32 | ✅ | ❌ | 内核检查拦截 |
| **8.0.20** | ✅ | **✅** | 无内核检查，正常运行 |

> 网上很多“Ubuntu 26.04 安装 MongoDB”的教程只验证到 `apt install` 成功就写“安装完成”，从不验证 `systemctl start`，照抄会踩坑。判断标准很简单：**文章有没有提内核 6.19+ 兼容性**。

---

## 三、解决方案：固定安装 8.0.20

### 1. 确认系统与内核

```bash
lsb_release -a
uname -r
```

Ubuntu 26.04 代号 `resolute`，但 **MongoDB 官方尚未发布 26.04 原生仓库**，需要借用 24.04（`noble`）的仓库。

### 2. 导入 GPG 密钥

```bash
sudo apt-get install -y gnupg curl ca-certificates

curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg
```

> 踩坑提示：`https://www.mongodb.org/static/pgp/server-9.0.asc` 是 404（9.0 的公钥没挂在这个路径），不要照抄 9.0 的教程。若遇到 `NO_PUBKEY`，说明密钥环是空的，重新执行上面这条命令即可。

### 3. 添加 APT 源（用 noble 仓库）

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt-get update
```

确认仓库可用、看看有哪些版本：

```bash
apt-cache policy mongodb-org-server
```

### 4. 安装指定版本 8.0.20

```bash
sudo apt-get install -y --allow-downgrades \
  mongodb-org=8.0.20 \
  mongodb-org-database=8.0.20 \
  mongodb-org-server=8.0.20 \
  mongodb-org-mongos=8.0.20 \
  mongodb-org-tools=8.0.20 \
  mongodb-mongosh
```

### 5. 启动并验证

```bash
sudo systemctl enable --now mongod
systemctl is-active mongod          # active

mongosh --quiet --eval "db.runCommand({ping:1})"
# { ok: 1 }
```

写入一条真实数据，确认读写正常：

```bash
mongosh --quiet test_crawler --eval \
  'db.books.insertMany([{title:"红楼梦",score:"9.6",price:"498.00元"}])'

mongosh --quiet test_crawler --eval \
  'db.books.find().forEach(printjson); "count: "+db.books.countDocuments()'

# 清理测试库
mongosh --quiet test_crawler --eval 'db.dropDatabase()'
```

### 6. 锁定版本（重要）

否则日后 `apt upgrade` 把版本升到 8.0.23+，服务会再次起不来：

```bash
sudo apt-mark hold mongodb-org mongodb-org-database mongodb-org-server \
  mongodb-org-mongos mongodb-org-tools

# 查看锁定状态
apt-mark showhold

# 以后官方适配了新内核，再解锁升级
# sudo apt-mark unhold mongodb-org*
```

---

## 四、安装图形客户端 Compass

```bash
cd /tmp
curl -sLO https://downloads.mongodb.com/compass/mongodb-compass_1.52.0_amd64.deb
sudo apt-get install -y ./mongodb-compass.deb
rm -f mongodb-compass_1.52.0_amd64.deb
```

启动：

```bash
mongodb-compass
```

连接串保持默认 `mongodb://localhost:27017`，点 Connect 即可。

> Compass **不支持中文界面**（官方没有 i18n）。想要中文 GUI 可以用 NoSQLBooster for MongoDB，官方带简体中文。

---

## 五、可选优化：省内存 + 启停联动

`mongod` 空载常驻约 200MB 内存。如果只是学习用途，可以让它**按需启动**：关掉开机自启，改用 Compass 启动时联动。

### 1. 关闭开机自启

```bash
sudo systemctl disable mongod
sudo systemctl stop mongod
```

### 2. 允许当前用户免密启停 mongod（polkit）

```bash
sudo tee /etc/polkit-1/rules.d/49-mongod-user.rules > /dev/null <<'EOF'
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units") {
        var unit = action.lookup("unit");
        if (unit == "mongod.service" && subject.user == "你的用户名" && subject.isInGroup("sudo")) {
            return polkit.Result.YES;
        }
    }
});
EOF

sudo systemctl restart polkit
```

> 该规则只放开 `mongod` 这一个服务单元，其他服务仍需密码。

### 3. Compass 包装脚本（开 Compass 自动开库，关 Compass 自动停库）

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/compass <<'EOF'
#!/bin/bash
# 启动 Compass 前先拉起 mongod，退出后自动关闭，节省常驻内存

systemctl start mongod 2>/dev/null

# 等待 mongod 就绪（最多 15 秒）
for i in $(seq 1 30); do
    mongosh --quiet --eval "db.runCommand({ping:1})" >/dev/null 2>&1 && break
    sleep 0.5
done

mongodb-compass "$@"

# Compass 关闭后停掉 mongod，释放内存
systemctl stop mongod 2>/dev/null
EOF

chmod +x ~/.local/bin/compass
```

创建用户级菜单入口（不修改系统自带文件）：

```bash
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/mongodb-compass.desktop <<'EOF'
[Desktop Entry]
Name=MongoDB Compass
GenericName=MongoDB Compass
Exec=/home/你的用户名/.local/bin/compass %U
Icon=mongodb-compass
Type=Application
Categories=GNOME;GTK;Utility;
StartupNotify=true
EOF

update-desktop-database ~/.local/share/applications/
```

之后从**应用菜单**点 MongoDB Compass 就会自动联动。注意：直接跑 `mongodb-compass` 命令不会触发联动，会报 `ECONNREFUSED 127.0.0.1:27017`。

---

## 六、Python 连接示例

```bash
pip install pymongo
```

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["crawler"]
db.books.insert_one({"title": "红楼梦", "score": "9.6", "price": "498.00元"})

for book in db.books.find():
    print(book)
```

---

## 七、风险声明与后续

1. **8.0.20 是变通方案，官方不支持在 kernel ≥ 6.19 上运行 MongoDB 8.x/9.x**，稳定性未经官方背书，请勿用于生产环境。
2. 内核检查是 MongoDB 针对已知崩溃风险加的**保护措施**，绕过它意味着承担潜在稳定性风险。本文仅用于个人学习环境验证。
3. 等官方修复 SERVER-121912 并发布适配版本后，应 `apt-mark unhold` 升级到受支持版本。
4. 数据无价：学习环境无所谓，真实数据请先备份再折腾。

## 参考链接

- 官方 issue：<https://jira.mongodb.org/browse/SERVER-121912>
- MongoDB 官方安装文档：<https://www.mongodb.com/docs/manual/administration/install-on-linux/>
- MongoDB Community 论坛讨论（8.x 与内核 6.19）：<https://www.mongodb.com/community/forums/t/mongodb-8-x-and-linux-kernel-6-19/337547>
- RoseHosting 教程（评论区提到需固定 8.0.20）：<https://www.rosehosting.com/blog/how-to-install-mongodb-on-ubuntu-26-04/>
