--- 
title: '【DDD-实践】社保公积金，工资计算功能的实现'
date: 2019-02-20 17:23:08
updated: 2019-02-20 17:23:04
categories:  
- practice
tags: 
- DDD
description: DDD
---

# 背景
> 人事云平台，需要实现计算工资，社保公积金的逻辑，同时给出 个人的工资条 和 企业需要支付的官费账单。


<!--more-->

- 各地的社保计算规则差距太大，进位规则，保留位数不一样。
- 一个城市还有很多种社保方案，每个方案的每个单项可能有单独的基数上下限，比例，固定值。

# 实现
IDEA 生成的 时序图（有删减）
![](https://user-images.githubusercontent.com/5629307/53080527-d8e4fa80-3533-11e9-8c99-f413ecfa8bb3.png)
最小的计算单元，包含了地区无关的最少的计算信息：
```java
InsuranceConfigDetail {
    @Id
    private Integer id;

    /**
     * 参保规则ID
     */
    @Column(name = "config_id")
    private Long configId;

    /**
     * 类型#1=养老，2=医疗，3=失业，4=共赏，5=生育，6=残保金，7=公积金，8=大病互助
     */
    @Column(name = "type_code")
    private Integer typeCode;

    /**
     * 付款方#1=公司，2=个人
     */
    @Column(name = "payer_code")
    private Integer payerCode;

    /**
     * 计算类型#1=rate（比例），2=value（固定值）
     */
    @Column(name = "rule_code")
    private Integer ruleCode;

    /**
     * 类别#1=社保，2=公积金
     */
    @Column(name = "cate_code")
    private Integer cateCode;

    @Column(name = "min_base")
    private BigDecimal minBase;

    @Column(name = "max_base")
    private BigDecimal maxBase;
    /**
     * 比例
     */
    private BigDecimal rate;
    /**
     * 固定值
     */
    private BigDecimal value;
}
```

调用链：
MSalaryService.cal(yearMonth, false, shouldDays, companyId);	// 计算工资
 |-SalaryDomain.calInsurance(InsuranceConfigDomain config) // 计算社保公积金
  |-InsuranceCalculator.cal(config, SalaryDomain);	// 社保公积金计算器
   |-SalaryDetail.cal(base, scale, mode); // 实际计算位置

```java
// com.zhijia.work.service.manage.MSalaryService#cal
List<SalaryDomain> domains = workers
.stream()
.filter(
	//过滤不需要计算的人
	// - 没有合同信息
	// - 员工已离职，且离职日期早于 社保公积金最后缴纳日期
)
.map(
	// 1. 冗余 worker_job_info 相关信息
	SalaryDomain record = SalaryDomain.create(worker);
	// 计算工资
)
.peek(
	// 计算社保公积金
	InsuranceConfigDomain config = idAndConfig.get(record.getCompanyInsuranceConfigId());
	try {
		record.calInsurance(config);
	} catch (NativeException e) {
		log.error(e.getMessage());
		throw new RollBackException(e.getCode(), e.getMessage());
	}
)
.peek(calOnCheck)   //加班 缺勤工资计算
.peek(SalaryDomain::calRest)   //计算 其他部分
.peek(record -> record.fillDate(yearMonth, today))  //时间计算
.peek(record -> {   //设置身份证信息
	/// ...
})
.collect(Collectors.toList());

```
```java
    public static WorkerInsuranceDto cal(InsuranceConfigDomain config, SalaryDomain info) throws NativeException {
        List<InsuranceConfigDetailDomain> configDetails = config.getDetails();
        List<InsuranceConfigDetailDomain> salaryDetails = configDetails.stream().
                map(configDetail -> {
                    // config => 工资记录
                    InsuranceConfigDetailDomain salaryDetail = new InsuranceConfigDetailDomain();
                    BeanUtils.copyProperties(configDetail, salaryDetail);
                    return salaryDetail;
                })
                .peek(salaryDetail -> {
                    BigDecimal base = info.getBaseByType(salaryDetail.getType());
                    InsuranceRule rule;
										// 按照 configId 和 payerCode 获取具体的计算规则。
                    if (salaryDetail.isCateInsurance()) {
											// 是社保
                        rule = InsuranceRule.matchInsuranceRule(salaryDetail.getConfigId(), salaryDetail.getPayerCode());
                    } else {
											// she 公积金
                        rule = InsuranceRule.matchFundRule(salaryDetail.getConfigId(), salaryDetail.getPayerCode());
                    }
                    Integer scale = rule.getScale();
                    RoundingMode mode = rule.getMode();
										// 调用 实际计算方法
                    BigDecimal result = salaryDetail.cal(base, scale, mode);
                    salaryDetail.setResult(result);
                }).collect(Collectors.toList());

        WorkerInsuranceDto dto = details2dto(salaryDetails);
        return dto;
    }
```
```java
public class InsuranceConfigDetailDomain extends InsuranceConfigDetail {
	    /**
     * @param employeeBase 单项基数
     * @param scale        进位 保留位数
     * @param mode         进位规则
     * @return
     */
    public BigDecimal cal(BigDecimal employeeBase, Integer scale, RoundingMode mode) {
        // 该员工不缴纳社保（公积金）？ =  社保（公积金）基数 == 0
        boolean notOpen = BigDecimal.ZERO.compareTo(employeeBase) == 0;
        if (notOpen) {
            return BigDecimal.ZERO;
        }

        BigDecimal base = actualBase(employeeBase);
        if (isRuleRate()) {
					// 是 比例
            BigDecimal result = base.multiply(getRate())
                    .divide(ONE_HUNDRED, scale, mode);
            return result;
        } else {
					// 是 值
            return getValue();
        }
    }
    /**
     * 最终生效的基数
     *
     * @param employeeBase
     * @return
     */
    private BigDecimal actualBase(BigDecimal employeeBase) {
        BigDecimal base = employeeBase
                .min(getMaxBase())
                .max(getMinBase());
        return base;
    }
		// 其他工具类
		groupByType
		groupByPayer
		isRuleRate
		isCateInsurance
}
```
InsuranceRule 保存了 各个地区社保计算规则：
一个元组，包括：
- configId：按照城市，地区，档位分类。
- 付款方：公司，个人。
- 进位规则：四舍五入，向上取整。
- 保留位数：1，2，3，4。
把所有相关的配置都封闭在这个类中，将改动限制到最小。

在 static block 中初始化，注册规则。
提供按照 规则 地区 ID 和 公司/个人 查询具体规则。

```java
public class InsuranceRule { 
    private Long configId;
		// 付款方
    private Integer payerCode;
		// 保留位数
    private Integer scale;
		// 进位规则
    private RoundingMode mode;
    static {
        /**
         * (10, '苏州', '市区'),
         (11, '苏州', '工业园区甲类'),
         (12, '苏州', '工业园区乙类'),
         (20, '北京', '市区'),
         (21, '北京', '农村'),
         (30, '深圳', '一档'),
         (31, '深圳', '二档'),
         (32, '深圳', '三档'),
         (40, '镇江', '市区'),
         (50, '上海', '默认'),
         (60, '南京', '默认'),
         (70, '合肥', '默认'),
         (80, '成都', '默认'),
         */
        //todo use Enum instead
        /**
         * 社保
         * 默认 四舍五入到两位
         */
        InsuranceRule baseRule = new InsuranceRule(0L, 0, 2, RoundingMode.HALF_UP);
        insuranceRules.add(baseRule);
        // 合肥 公司 四舍五入 3位
        insuranceRules.add(new InsuranceRule(70L, 1, 3, RoundingMode.HALF_UP));
        // 上海 公司 四舍五入 4位
        insuranceRules.add(new InsuranceRule(50L, 1, 4, RoundingMode.HALF_UP));
        // 上海 个人 向上取整 1位
        insuranceRules.add(new InsuranceRule(50L, 2, 1, RoundingMode.UP));

        /**
         * 公积金
         * 默认 四舍五入到整数
         */
        InsuranceRule baseRule1 = new InsuranceRule(0L, 0, 0, RoundingMode.UP);
        // 苏州  园区  个人和公司 四舍五入 2位
        fundRules.add(baseRule1);
        fundRules.add(new InsuranceRule(11L, 1, 2, RoundingMode.HALF_UP));
        fundRules.add(new InsuranceRule(11L, 2, 2, RoundingMode.HALF_UP));
        fundRules.add(new InsuranceRule(12L, 1, 2, RoundingMode.HALF_UP));
        fundRules.add(new InsuranceRule(12L, 2, 2, RoundingMode.HALF_UP));
    }
    public static InsuranceRule matchInsuranceRule(Long configId, Integer payerCode) {
        Optional<InsuranceRule> first = insuranceRules.stream()
                .filter(rule -> configId.equals(rule.configId) && payerCode.equals(rule.payerCode))
                .findFirst();
        return first.orElse(insuranceRules.get(0));
    }

    public static InsuranceRule matchFundRule(Long configId, Integer payerCode) {
        Optional<InsuranceRule> first = fundRules.stream()
                .filter(rule -> configId.equals(rule.configId) && payerCode.equals(rule.payerCode))
                .findFirst();
        return first.orElse(fundRules.get(0));
    }
}
```

计算工资其他部分的核心代码：
也就是 上面 `calRest` 中的实现。
通过一个顺序严格的链式调用，计算工资的各个部分，
规则：
- 应发工资=基本工资+其他工资+补贴+加班工资-缺勤扣款
- 应税工资=应发工资-社保公积金_个人部分
- 实际工资=应税工资-个人所得税
- 官费=应发工资+社保公积金_公司部分

同时，计算五险一金的调用也是在 SalaryDomain 中。
- 计算社保公积金
- 填充各个单项的值
- 计算社保公积金_个人部分 总和
- 计算社保公积金_公司部分 总和
```java
public class SalaryDomain extends WorkerSalaryRecord
        implements DomainMixin<WorkerSalaryRecord, SalaryDomain>,
        InsuranceMixin,
        LoggerMixin {
		/**
     * 计算五险一金
     *
     * @return
     */
    public SalaryDomain calInsurance(InsuranceConfigDomain config) throws NativeException {
        logger().debug("calInsurance");

        if (config == null) {
            throw new NativeException("[salary.calInsurance]社保公积金配置为空");
        }
        WorkerInsuranceDto insuranceDto = InsuranceCalculator.cal(config, this);
        // 2. 五险一金明细
        fillInsuranceDetails(insuranceDto);
        return this;
    }
    /**
     * 计算
     * 1. 应发工资
     * 2. 应税工资
     * 3. 个人所得税
     * 4. 实发工资
     * 5. 官费
     * <p>
     * required:
     * getBasicWage
     * getOtherWage
     * getTrafficAllow
     * getMealAllow
     * getHighTempAllow
     * getOtherAllow
     * getOvertimeWage
     * getAbsenceWage
     * getPersonTotal
     * getCompTotal
     *
     * @return
     */
    public SalaryDomain calRest() {
        return calMustWage()  //应发工资
                .calMustTaxWage()//应税工资
                .calTax()   //个人所得税
                .calActualWage()   //实际工资
                .calFeeTotal()    //计算官费=应发工资+社保公积金_公司部分
                ;
    }
				}
```