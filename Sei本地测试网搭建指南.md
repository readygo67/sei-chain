# Sei 本地测试网搭建指南

## 📋 目录
- [概述](#概述)
- [环境准备](#环境准备)
- [快速搭建](#快速搭建)
- [手动搭建](#手动搭建)
- [脚本详解](#脚本详解)
- [测试账户管理](#测试账户管理)
- [常用命令](#常用命令)
- [故障排除](#故障排除)

---

## 概述

本指南将帮助你在本地机器上运行一个 Sei 单节点测试网，包括完整的初始化、配置和启动流程。

### 🎯 目标
- 搭建本地 Sei 测试网
- 创建测试账户
- 配置快速出块（2秒）
- 启用 Sei 核心特性（OCC、SeiDB）

---

## 环境准备

### 系统要求
- **操作系统**: macOS / Linux
- **Go**: 1.18+
- **Python**: 3.x
- **内存**: 建议 8GB+
- **存储**: 建议 10GB+ 可用空间

### 依赖检查
```bash
# 检查 Go 版本
go version

# 检查 Python 版本
python3 --version

# 检查 jq（JSON 处理工具）
jq --version
```

---

## 快速搭建

### 🚀 方案一：自动化脚本（推荐）

**步骤 1: 安装 seid**
```bash
cd /Users/luokeep/Code/github.com/readygo67/sei-chain
make install
```

**步骤 2: 运行初始化脚本**
```bash
NO_RUN=0 bash scripts/initialize_local_chain.sh
```

**步骤 3: 验证节点运行**
```bash
# 检查节点状态
~/go/bin/seid status

# 查询账户余额
~/go/bin/seid query bank balances $(~/go/bin/seid keys show admin -a --keyring-backend test)
```

---

## 手动搭建

### 🔧 方案二：逐步配置（了解细节）

**步骤 1: 清理旧配置**
```bash
rm -rf ~/.sei
```

**步骤 2: 初始化链**
```bash
~/go/bin/seid init demo --chain-id sei-chain
```

**步骤 3: 创建管理员密钥**
```bash
~/go/bin/seid keys add admin --keyring-backend test
```

**步骤 4: 添加创世账户**
```bash
~/go/bin/seid add-genesis-account $(~/go/bin/seid keys show admin -a --keyring-backend test) 100000000000000000000usei,100000000000000000000uusdc,100000000000000000000uatom --keyring-backend test
```

**步骤 5: 创建创世交易**
```bash
~/go/bin/seid gentx admin 7000000000000000usei --chain-id sei-chain --keyring-backend test
```

**步骤 6: 添加验证人信息**
```bash
KEY=$(jq '.pub_key' ~/.sei/config/priv_validator_key.json -c)
jq '.validators = [{}]' ~/.sei/config/genesis.json > ~/.sei/config/tmp_genesis.json
jq '.validators[0] += {"power":"7000000000"}' ~/.sei/config/tmp_genesis.json > ~/.sei/config/tmp_genesis_2.json
jq '.validators[0] += {"pub_key":'$KEY'}' ~/.sei/config/tmp_genesis_2.json > ~/.sei/config/tmp_genesis_3.json
mv ~/.sei/config/tmp_genesis_3.json ~/.sei/config/genesis.json && rm ~/.sei/config/tmp_genesis.json && rm ~/.sei/config/tmp_genesis_2.json
```

**步骤 7: 创建测试账户**
```bash
python3 loadtest/scripts/populate_genesis_accounts.py 20 loc
```

**步骤 8: 收集创世交易**
```bash
~/go/bin/seid collect-gentxs
```

**步骤 9: 配置 keyring backend**
```bash
~/go/bin/seid config keyring-backend test
```

**步骤 10: 启动节点**
```bash
~/go/bin/seid start --chain-id sei-chain
```

---

## 脚本详解

### initialize_local_chain.sh 逐行解析

#### 1. 环境检测（第 6-11 行）
```bash
PYTHON_CMD=python3
if ! command -v $PYTHON_CMD &> /dev/null
then
    PYTHON_CMD=python
fi
```
**作用**: 检测 Python 版本，优先使用 python3，回退到 python

#### 2. 清理和编译（第 30-34 行）
```bash
rm -rf ~/.sei
echo "Building..."
make install
```
**作用**: 删除旧配置，编译安装 seid 二进制文件

#### 3. 初始化链（第 36-37 行）
```bash
~/go/bin/seid init demo --chain-id sei-chain
~/go/bin/seid keys add $keyname --keyring-backend test
```
**作用**: 初始化节点配置，创建管理员密钥

#### 4. 添加创世账户（第 39 行）
```bash
~/go/bin/seid add-genesis-account $(~/go/bin/seid keys show $keyname -a --keyring-backend test) 100000000000000000000usei,100000000000000000000uusdc,100000000000000000000uatom --keyring-backend test
```
**作用**: 给 admin 账户分配初始余额（100万亿 SEI、USDC、ATOM）

#### 5. 创建创世交易（第 41 行）
```bash
~/go/bin/seid gentx $keyname 7000000000000000usei --chain-id sei-chain --keyring-backend test
```
**作用**: 生成验证人的创世交易（质押 70亿 SEI）

#### 6. 添加验证人信息（第 43-47 行）
```bash
KEY=$(jq '.pub_key' ~/.sei/config/priv_validator_key.json -c)
jq '.validators = [{}]' ~/.sei/config/genesis.json > ~/.sei/config/tmp_genesis.json
jq '.validators[0] += {"power":"7000000000"}' ~/.sei/config/tmp_genesis.json > ~/.sei/config/tmp_genesis_2.json
jq '.validators[0] += {"pub_key":'$KEY'}' ~/.sei/config/tmp_genesis_2.json > ~/.sei/config/tmp_genesis_3.json
mv ~/.sei/config/tmp_genesis_3.json ~/.sei/config/genesis.json && rm ~/.sei/config/tmp_genesis.json && rm ~/.sei/config/tmp_genesis_2.json
```
**作用**: 手动添加验证人信息到 genesis.json（Sei 特有要求）

#### 7. 创建测试账户（第 51 行）
```bash
python3 loadtest/scripts/populate_genesis_accounts.py 20 loc
```
**作用**: 批量创建 20 个测试账户，每个获得 10 亿 SEI

#### 8. 调整链参数（第 55-65 行）
```bash
# 治理参数 - 快速投票
cat ~/.sei/config/genesis.json | jq '.app_state["gov"]["deposit_params"]["max_deposit_period"]="60s"' > ~/.sei/config/tmp_genesis.json && mv ~/.sei/config/tmp_genesis.json ~/.sei/config/genesis.json

# 预言机白名单
cat ~/.sei/config/genesis.json | jq '.app_state["oracle"]["params"]["whitelist"]=[{"name": "ueth"},{"name": "ubtc"},{"name": "uusdc"},{"name": "uusdt"},{"name": "uosmo"},{"name": "uatom"},{"name": "usei"}]' > ~/.sei/config/tmp_genesis.json && mv ~/.sei/config/tmp_genesis.json ~/.sei/config/genesis.json

# 区块限制
cat ~/.sei/config/genesis.json | jq '.consensus_params["block"]["max_gas"]="35000000"' > ~/.sei/config/tmp_genesis.json && mv ~/.sei/config/tmp_genesis.json ~/.sei/config/genesis.json
```
**作用**: 调整各种链参数，优化测试体验

#### 9. 启用 Sei 特性（第 81-84 行）
```bash
sed -i.bak -e 's/# concurrency-workers = .*/concurrency-workers = 500/' $APP_TOML_PATH
sed -i.bak -e 's/occ-enabled = .*/occ-enabled = true/' $APP_TOML_PATH
sed -i.bak -e 's/sc-enable = .*/sc-enable = true/' $APP_TOML_PATH
sed -i.bak -e 's/ss-enable = .*/ss-enable = true/' $APP_TOML_PATH
```
**作用**: 启用 OCC（乐观并发控制）和 SeiDB（高性能存储）

#### 10. 配置区块时间（第 100-125 行）
```bash
# Linux 配置
sed -i 's/timeout_prevote =.*/timeout_prevote = "2000ms"/g' $CONFIG_PATH
sed -i 's/timeout_precommit =.*/timeout_precommit = "2000ms"/g' $CONFIG_PATH
sed -i 's/timeout_commit =.*/timeout_commit = "2000ms"/g' $CONFIG_PATH

# macOS 配置
sed -i '' 's/unsafe-propose-timeout-override =.*/unsafe-propose-timeout-override = "2s"/g' $CONFIG_PATH
sed -i '' 's/unsafe-vote-timeout-override =.*/unsafe-vote-timeout-override = "2s"/g' $CONFIG_PATH
```
**作用**: 设置 2 秒区块时间，加快测试

---

## 测试账户管理

### 📄 查看测试账户

#### 方法 1: 使用便捷脚本
```bash
# 创建查看脚本
cat > ~/view_test_accounts.sh << 'EOF'
#!/bin/bash
echo "==================================="
echo "   Sei 测试账户列表 (20个账户)"
echo "==================================="
echo ""
for i in {0..19}; do
    addr=$(jq -r '.address' ~/test_accounts/ta$i.json 2>/dev/null)
    if [ ! -z "$addr" ]; then
        echo "[$i] ta$i: $addr"
    fi
done
echo ""
echo "余额: 每个账户 1,000,000,000 SEI"
echo "查看助记词: cat ~/test_accounts/ta0.json"
echo "使用账户: seid keys show ta0 --keyring-backend test"
EOF
chmod +x ~/view_test_accounts.sh

# 运行脚本
~/view_test_accounts.sh
```

#### 方法 2: 查看单个账户
```bash
# 查看完整信息（包含助记词）
cat ~/test_accounts/ta0.json | jq .

# 只查看地址
jq -r '.address' ~/test_accounts/ta0.json

# 只查看助记词
jq -r '.mnemonic' ~/test_accounts/ta0.json
```

#### 方法 3: 批量查看
```bash
# 列出所有地址
for i in {0..19}; do 
    echo "ta$i: $(jq -r '.address' ~/test_accounts/ta$i.json)"
done
```

### 🔑 使用 seid 命令查看
```bash
# 列出所有密钥
~/go/bin/seid keys list --keyring-backend test

# 查看特定账户详情（包含 EVM 地址）
~/go/bin/seid keys show ta0 --keyring-backend test

# 只显示 Sei 地址
~/go/bin/seid keys show ta0 -a --keyring-backend test
```

### 💰 查看余额
```bash
# 查询单个账户余额
~/go/bin/seid query bank balances $(~/go/bin/seid keys show ta0 -a --keyring-backend test)

# 查询所有测试账户余额
for i in {0..19}; do
    addr=$(~/go/bin/seid keys show ta$i -a --keyring-backend test)
    echo "ta$i ($addr):"
    ~/go/bin/seid query bank balances $addr --chain-id sei-chain
done
```

### 📊 导出为 CSV
```bash
# 创建 CSV 文件
echo "序号,账户名,Sei地址,EVM地址" > ~/test_accounts.csv
for i in {0..19}; do
    addr=$(jq -r '.address' ~/test_accounts/ta$i.json)
    evm=$(~/go/bin/seid keys show ta$i --keyring-backend test 2>/dev/null | grep evm_address | awk '{print $2}')
    echo "$i,ta$i,$addr,$evm" >> ~/test_accounts.csv
done
echo "✅ 已导出到 ~/test_accounts.csv"
```

---

## 常用命令

### 🚀 节点管理
```bash
# 启动节点
~/go/bin/seid start --chain-id sei-chain

# 检查节点状态
~/go/bin/seid status

# 查看节点信息
~/go/bin/seid query node-info

# 停止节点
pkill seid
```

### 💸 发送交易
```bash
# 发送代币
~/go/bin/seid tx bank send ta0 \
  $(~/go/bin/seid keys show ta1 -a --keyring-backend test) \
  1000usei \
  --chain-id sei-chain \
  --keyring-backend test \
  --fees 4000usei \
  --yes

# 查询交易
~/go/bin/seid query tx <tx_hash>
```

### 🔍 查询操作
```bash
# 查询区块
~/go/bin/seid query block <height>

# 查询账户
~/go/bin/seid query auth account <address>

# 查询验证人
~/go/bin/seid query staking validators

# 查询余额
~/go/bin/seid query bank balances <address>
```

### 📊 链信息
```bash
# 查看创世文件
cat ~/.sei/config/genesis.json | jq .

# 查看配置
cat ~/.sei/config/config.toml
cat ~/.sei/config/app.toml

# 查看链 ID
~/go/bin/seid config chain-id
```

---

## 故障排除

### ❌ 常见问题

#### 1. 节点无法启动 - 没有验证人
**症状**: 节点启动后卡住，没有出块
**原因**: genesis.json 中缺少 validators 信息
**解决**: 手动添加验证人信息
```bash
KEY=$(jq '.pub_key' ~/.sei/config/priv_validator_key.json -c)
jq '.validators = [{}]' ~/.sei/config/genesis.json > ~/.sei/config/tmp_genesis.json
jq '.validators[0] += {"power":"7000000000"}' ~/.sei/config/tmp_genesis.json > ~/.sei/config/tmp_genesis_2.json
jq '.validators[0] += {"pub_key":'$KEY'}' ~/.sei/config/tmp_genesis_2.json > ~/.sei/config/tmp_genesis_3.json
mv ~/.sei/config/tmp_genesis_3.json ~/.sei/config/genesis.json
```

#### 2. Python 命令不存在
**症状**: `python3: command not found`
**解决**: 安装 Python 3
```bash
# macOS
brew install python3

# Ubuntu
sudo apt-get install python3
```

#### 3. jq 命令不存在
**症状**: `jq: command not found`
**解决**: 安装 jq
```bash
# macOS
brew install jq

# Ubuntu
sudo apt-get install jq
```

#### 4. 端口被占用
**症状**: `bind: address already in use`
**解决**: 杀死占用端口的进程
```bash
# 查找占用端口的进程
lsof -i :26657
lsof -i :1317
lsof -i :8545

# 杀死进程
kill -9 <PID>
```

#### 5. 权限问题
**症状**: `permission denied`
**解决**: 检查文件权限
```bash
# 检查权限
ls -la ~/.sei/config/

# 修改权限
chmod 644 ~/.sei/config/genesis.json
chmod 600 ~/.sei/config/priv_validator_key.json
```

### 🔧 调试技巧

#### 启用详细日志
```bash
# 启动时启用 trace
~/go/bin/seid start --chain-id sei-chain --trace

# 启用竞态检测
GORACE="log_path=/tmp/race/seid_race" ~/go/bin/seid start --trace --chain-id sei-chain
```

#### 检查配置文件
```bash
# 验证 genesis.json 格式
cat ~/.sei/config/genesis.json | jq . > /dev/null && echo "✅ Valid JSON" || echo "❌ Invalid JSON"

# 检查验证人数量
cat ~/.sei/config/genesis.json | jq '.validators | length'

# 检查账户数量
cat ~/.sei/config/genesis.json | jq '.app_state.auth.accounts | length'
```

#### 重置链
```bash
# 完全重置
rm -rf ~/.sei
bash scripts/initialize_local_chain.sh
```

---

## 📋 重要端口

| 端口 | 服务 | 说明 |
|------|------|------|
| 26656 | P2P | 节点间通信 |
| 26657 | RPC | Tendermint RPC |
| 1317 | REST API | Cosmos SDK REST API |
| 9090 | gRPC | Cosmos SDK gRPC |
| 8545 | EVM RPC | 以太坊兼容 RPC |

---

## 📊 账户信息汇总

### 管理员账户
- **名称**: admin
- **地址**: `sei18x7lvug52g8qmslz4ukauplzr5njxny09uchnx`
- **余额**: 100,000,000,000,000 SEI + USDC + ATOM

### 测试账户 (ta0-ta19)
- **数量**: 20 个
- **余额**: 每个 1,000,000,000 SEI
- **文件位置**: `~/test_accounts/ta*.json`
- **用途**: 负载测试、开发测试

### 文件结构
```
~/.sei/
├── config/
│   ├── genesis.json          # 创世文件
│   ├── config.toml          # 共识配置
│   ├── app.toml             # 应用配置
│   └── priv_validator_key.json  # 验证人私钥
├── data/                    # 区块链数据
└── keyring-test/           # 测试密钥环

~/test_accounts/
├── ta0.json                # 测试账户 0
├── ta1.json                # 测试账户 1
└── ...                     # ta2.json 到 ta19.json
```

---

## ⚠️ 安全提醒

- **测试环境专用**: 这些配置仅用于本地测试
- **明文存储**: test keyring 模式下私钥以明文存储
- **不要用于生产**: 绝对不要在生产环境或主网使用
- **定期清理**: 测试完成后及时清理敏感数据

---

## 🎉 完成！

现在你已经成功搭建了 Sei 本地测试网！你可以：

1. ✅ 使用 20 个测试账户进行开发
2. ✅ 体验 2 秒快速出块
3. ✅ 测试 Sei 的 OCC 和 SeiDB 特性
4. ✅ 部署和测试智能合约
5. ✅ 进行负载测试

如有问题，请参考故障排除部分或查看 Sei 官方文档。

---

*文档生成时间: $(date)*
*Sei 版本: v6.1.12*
