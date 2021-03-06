# 资金管理
* 含业务收支的账户系统

## 术语
| 名称 | 说明 |
| :----: | ---- |
| 账户 |  |
| 账户 - 金额(结余) | 金额是动态的，和明细联动。和账单无关 |
| 账户 - 明细(收支) | 如一个账单的收入 |
| 账单 | 由多个业务收支统计组成 |
| 业务收支 | 如服务收入，垫款等 |

# 设计
## 账户

### 入账出账
* 账户的金额转入、转出，要有对应的账户明细记录。转入时，账单明细的金额为正数；转出时，账单明细的金额为负数
### 提现
* 是一种出账
* 提现方式： 手动，自动
* 手动，自动都有最低额度限制
* 用户提现资格：手动，账户的金额大于最低额度限制

## 业务收支

### 结构
1. 类型：正常，[冲正](https://zhidao.baidu.com/question/449565606.html)
1. 业务号
1. 冲正业务号：正常收支的是NULL，冲正收支的是正常收支的业务号
1. 入账状态
  * 正常收支是多态的，冲正收支的状态只有确认成功
  * 正常收支的状态实例

| 状态 | 对应服务状态 | 说明 |
| :----: | ---- | ---- |
| 待确认 | 服务中 | 如交易已支付 |
| 确认成功 | 服务完成 | 如交易完成 |
| 确认失败 | 服务取消 | 如交易支付后退款 |

### 实例

| 业务 | 业务操作 | 类型 | 收支金额 | 业务号 | 冲正业务号 |
| :----: | ---- | ---- |
| 服务 | 完成 | 正常 | 正 | 工单 | 无 |
| 服务 | 撤销 | 冲正 | 负 | 工单撤销单 | 工单 |
| 交易(提成) | 完成 | 正常 | 正 | 订单 | 无 |
| 交易(提成) | 完成后退货 | 冲正 | 负 | 退货单 | 订单 |
| 支付(垫资) | 支付 | 正常 | 负 | 支付单 | 无 |
| 支付(垫资) | 退款 | 冲正 | 正 | 退款单 | 支付单 |

### 记录生成方案
1. 即时
  * 业务系统在业务发生时发起
1. 定期检查
  * 业务系统周期性发起，检查是否所有的业务行为都有业务收支了，没有就加上

### 结账判断
* 账单号有值是已结账

## 账单
* 历史单据，无期初期末
* 月初把上月开始的每项业务(如所有未结账的收支)做统计。比如本月通话时间是100分钟，但不会有每次通话记录的时间详细。
* 账户金额可作为辅助信息显示在单据上

### 生成
* 启动：下个账期的期初
* 内容
  1. 未结账的服务收入：确认成功 && 账单号为空 && 日期<=本账期最后日期
  1. 未结账的业务收支：类似服务收入，有收入有支出
* 明细格式

| 名称 | 单位值 | 金额 |
| :----: | ---- | ---- |
| 电话 | 100分钟 | 30 |
| 4G | 50M | 10 |

### 付款
* 账户有收支，金额会改变
* 平台和商户双方都需要操作付款和确认收款(付款流水号)，收付款基于账户而不是账单。[流程](#账户收付款)
* 支持账户自动扣款

# 流程
## 入账
1. 把需要转入的金额转入账户，比如账单
1. 生成账户明细
1. 更改账户金额，金额=金额+转入的金额
1. 转入账户完成

## 出账
1. 把金额转出账户，比如提现
1. 生成账户明细，金额为负
1. 转出账户完成

## 业务收支的账期结算
1. 账户到账期时（比如滴滴司机是日结）
1. 计算本账期内业务收支金额，生成账单
1. 账单金额入账到账户(可能是负)

## 账户收付款
1. A方查看账户金额，如为负数则打款
1. A方公司打款，在系统内创建个打款记录（输入付款流水号和金额）
1. B方查看打款记录，确认收款
1. 确认后，调整商户账户的金额：A是商户，则是收入；A是平台，则是支出。

## 提现
1. 手动提现：当账户的金额大于最低额度限制，用户可以申请提现，生成申请提现单，平台财务人员根据申请提现单进行打款
1. 自动提现：每月初，当账户的金额大于最低额度限制，则自动生成申请提现单，平台财务人员根据申请提现单进行打款

### 手动提现
1. 申请提现
1. 判断提现资格
1. 生成提现申请单，交易状态：提现申请中
1. 生成账户明细。改变账户金额，账户金额=账户金额-提现金额
1. 财务人员打款（运营人员excel导出提现记录，交给财务人员打款。打款后导入后台，更改打款状态）
1. 设置流水号，状态为提现成功

### 自动提现
1. 每月初查询每个账户
1. 符合提现
1. 生成提现申请单，交易状态：提现申请中
1. 生成账户明细。改变账户金额，账户金额=账户金额-提现金额
1. 财务人员打款（运营人员excel导出提现记录，交给财务人员打款。打款后导入后台，更改打款状态）
1. 打款状态更改
1. 打款成功。财务人员设置流水号。状态为提现成功

# 参考
## 期初期末报表方案
### 生成流程
本账期开始时生成上期账单

1. 将上个账期的期末设置成本账期的期初
1. 计算本账期的账户进出
1. 生成本账期的期末和账单
### 实例

```
第一个月：
期初：0元
入账：服务收入所得100元
出账：平台服务费10元；提现70元
期末：20元 = 0+100+(-10)+(-70)

第二个月：
期初：20元
入账：服务收入所得200元
出账：平台服务费10元；提现40元
期末：170元 = 20+200+(-10)+(-40)
```
