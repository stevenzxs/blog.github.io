---
title: 分页同步数据
top: false
cover: false
toc: true
mathjax: true
date: 2024-12-22 12:25:39
password:
summary:
tags:
categories:
---

```java
long startTime=System.currentTimeMillis();   //获取开始时间
long totalNum=0;
CommonPage vo = new CommonPage();
while(true) {
    List<ProjectMap> orgList = projectMapMapper.selectOrgAll(vo);
    List<String> orgListProjectNames = orgList.stream().filter(t -> {
        return StringUtils.isNotBlank(t.getProjectName());
    }).map(st -> {
        return st.getProjectName().toUpperCase(Locale.ROOT);
    }).collect(Collectors.toList());
    if (orgList.isEmpty()) {
        break;
    } else {
        System.out.println("当前页ORG处理数据(条)：" + orgList.size());
        CommonPage distVo = new CommonPage();
        while(true) {
            List<ProjectMap> distListTemp = projectMapMapper.selectDistAll(distVo);
            List<ProjectMap> distList = distListTemp.stream().filter(tt -> {
                return StringUtils.isNotBlank(tt.getProjectName()) && orgListProjectNames.contains(tt.getProjectName().toUpperCase(Locale.ROOT));
            }).collect(Collectors.toList());
            if (distList.isEmpty()) {
                break;
            } else {
                System.out.println("当前页DIST处理数据(条)：" + distList.size());
                List<String> distListProductNames = projectMapMapper.selectProductTradeAll(distList);
                if (!distListProductNames.isEmpty()) {
                    Integer pageSize = 2000;
                    Integer pageNo = 1;
                    while(true) {
                        List<String> subList = distListProductNames.stream().skip((pageNo-1)*pageSize).limit(pageSize).
                                collect(Collectors.toList());
                        if (subList.isEmpty()) {
                            break;
                        }
                        projectMapMapper.batchUpdateProduct(subList);
                        pageNo = pageNo +1;
                    }
                }
                projectMapMapper.batchUpdateProductTrade(distList);
                int row = projectMapMapper.batchUpdateProject(distList);
                if (row > 0) {
                    totalNum += row;
                }
            }
            distVo.nexPage();
        }
    }
    vo.nexPage();
}

System.out.println("总同步数据(条)：" + totalNum);
long endTime=System.currentTimeMillis(); //获取结束时间
float runTime = (endTime-startTime) /1000;
System.out.println("程序运行时间： "+ runTime + "秒");
```

```java
package vip.xiaonuo.common.page;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Data
public class CommonPage {
    private Integer start=0;
    private Integer end=2000;
    private Integer num=2000;

    public void nexPage() {
        this.start = this.start + this.num;
        this.end = this.end + this.num;
    }
    public void prePage() {
        this.start = this.start - this.num;
        this.end = this.end - this.num;
        if (this.start < 0) {
            this.start = 0;
        }
        if (this.end < 2000) {
            this.end = 2000;
        }
    }
}
```

```java
/*
 * Copyright [2022] [https://www.xiaonuo.vip]
 *
 * Snowy采用APACHE LICENSE 2.0开源协议，您在使用过程中，需要注意以下几点：
 *
 * 1.请不要删除和修改根目录下的LICENSE文件。
 * 2.请不要删除和修改Snowy源码头部的版权声明。
 * 3.本项目代码可免费商业使用，商业使用请保留源码和相关描述文件的项目出处，作者声明等。
 * 4.分发源码时候，请注明软件出处 https://www.xiaonuo.vip
 * 5.不可二次分发开源参与同类竞品，如有想法可联系团队xiaonuobase@qq.com商议合作。
 * 6.若您的项目无法满足以上几点，需要更多功能代码，获取Snowy商业授权许可，请在官网购买授权，地址为 https://www.xiaonuo.vip
 */
package vip.xiaonuo.auth.modular.third.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;
import vip.xiaonuo.auth.modular.third.entity.AuthThirdUser;
import vip.xiaonuo.auth.modular.third.entity.ProjectMap;
import vip.xiaonuo.common.page.CommonPage;

import java.util.List;

/**
 * 第三方登录Mapper接口
 *
 * @author xuyuxiang
 * @date 2022/7/9 14:35
 */
public interface ProjectMapMapper extends BaseMapper<ProjectMap> {
    @Select({"<script>",
            " select * FROM project_map",
            " limit #{vo.start}, #{vo.end}",
            "</script>"})
    List<ProjectMap> selectDistAll(@Param("vo") CommonPage vo);


    @Select({"<script>",
            " select * FROM project_dict",
            " limit #{vo.start}, #{vo.end}",
            "</script>"})
    List<ProjectMap> selectOrgAll(@Param("vo") CommonPage vo);


    @Update({"<script>",
            " update project_map",
            " SET customer_name = CASE",
            " <foreach collection=\"voList\" item=\"item\" index=\"index\">",
            " WHEN project_id = #{item.projectId} THEN concat_ws('_',#{item.city},#{index})",
            " </foreach>",
            " END,",
            " trade = CASE",
            " <foreach collection=\"voList\" item=\"item\" index=\"index\">",
            " WHEN project_id = #{item.projectId} THEN concat_ws('_',#{item.city},#{index})",
            " </foreach>",
            " END",
            " WHERE project_id IN",
            " <foreach collection=\"voList\" index=\"index\" item=\"item\" open=\"(\" separator=\",\" close=\")\">",
            " #{item.projectId}",
            " </foreach>",
            "</script>"})
    int batchUpdateProject(@Param("voList") List<ProjectMap> voList);

    @Update({"<script>",
            " update product_trade",
            " SET create_user = CASE",
            " <foreach collection=\"voList\" item=\"item\" index=\"index\">",
            " WHEN project_id= #{item.projectId} THEN concat_ws('_','a',#{index})",
            " </foreach>",
            " END,",
            " update_user = CASE",
            " <foreach collection=\"voList\" item=\"item\" index=\"index\">",
            " WHEN project_id = #{item.projectId} THEN concat_ws('_','b',#{index})",
            " </foreach>",
            " END",
            " WHERE project_id IN",
            " <foreach collection=\"voList\" index=\"index\" item=\"item\" open=\"(\" separator=\",\" close=\")\">",
            " #{item.projectId}",
            " </foreach>",
            "</script>"})
    int batchUpdateProductTrade(@Param("voList") List<ProjectMap> voList);


    @Select({"<script>",
            " select product_id FROM product_trade",
            " where project_id in ",
            " <foreach collection=\"voList\" index=\"index\" item=\"item\" open=\"(\" separator=\",\" close=\")\">",
            " #{item.projectId}",
            " </foreach>",
            "</script>"})
    List<String> selectProductTradeAll(@Param("voList") List<ProjectMap> voList);

    @Update({"<script>",
            " update product_map",
            " SET create_user = CASE",
            " <foreach collection=\"voList\" item=\"item\" index=\"index\">",
            " WHEN product_id= #{item} THEN concat_ws('_','a',#{index})",
            " </foreach>",
            " END,",
            " update_user = CASE",
            " <foreach collection=\"voList\" item=\"item\" index=\"index\">",
            " WHEN product_id = #{item} THEN concat_ws('_','b',#{index})",
            " </foreach>",
            " END",
            " WHERE product_id IN",
            " <foreach collection=\"voList\" index=\"index\" item=\"item\" open=\"(\" separator=\",\" close=\")\">",
            " #{item}",
            " </foreach>",
            "</script>"})
    int batchUpdateProduct(@Param("voList") List<String> voList);
}
```