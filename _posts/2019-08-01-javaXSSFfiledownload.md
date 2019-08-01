---
layout: post
title: java自定义excel,文件下载
date: 2019-07-31
Author: Applefruits
tags: [Java,Excel,XSSF,FileDownload,CustomTemplate]
comments: true
---
>本文主要介绍spring boot环境下，利用Apache POI操作Excel,实现Excel文件的在线下载。

首先，我们在这里介绍一下关于ApachePOI中关于HSSF，XSSF和SXSSF的区别。

HSSF是POI工程对Excel 97(-2007)文件操作的纯Java实现，使用的 **.xls** 做结尾的文件  
XSSF是POI工程对Excel 2007以后版本 (.xlsx)文件操作的纯Java实现，使用 **.xlsx** 做结尾的文件   
从POI 3.8版本开始，提供了一种基于XSSF的低内存占用的API----SXSSF  

因为Excel版本的不断升级，就目前使用而言，一般使用较多的是XSSF，而网上大部分教程还停留在老版本（HSSF）；SXSSF由于使用用户较少，还有不少不适用户使用习惯的坑未填，对用普通开发者来说较为复杂，所以只要不是对性能有较强要求的用户，不建议使用。接下来就主要介绍一下XSSF的使用：

---
1. 首先根据需求创建模板文件，在模板文件中设置好要输出的列名（表头），注意要  **和sql检索出来的列名一致**![](/img/excel模板.png "excel模板")  
2. 上代码
```
@Override
public void reportsExcelPrint(HttpServletResponse response) {
List<Map<String,Object>> reportList = null;
String projectName="";
//检索数据库数据--用来赋值Excel
reportList = projectMapper.selByValue(projectName);
//模板文件路径
String modalUrl="template/项目周报.xlsx";
//sheet名称
String sheetName="项目周报";
// 对应数据库列名--和模板设置的列一致
String[] colKeys={"weekNumber","projectName"};
exportFile(reportList,colKeys,modalUrl,sheetName,response);
}
```
```
private void exportFile(List<Map<String, Object>> reportList, String[] colKeys, String modalUrl, String sheetName, HttpServletResponse response) {
    OutputStream out = null;
    try {
        Map<String, Object> results = new HashMap<>();
        results.put("talentFlows", "");
        // 拿到模板文件
        InputStream in = getClass().getClassLoader().getResourceAsStream(modalUrl);
        //新建workbook
        tpWorkbook = new XSSFWorkbook(in);
        //填入数据
        sheet = tpWorkbook.getSheet(sheetName);
        if (reportList != null) {
            //设置表格数据
            setCellData(reportList, colKeys);
        }
        //输出流
        out = response.getOutputStream();
        response.reset();
        response.setHeader("content-disposition",
                "attachment;filename=" + new String((sheetName).getBytes("gb2312"), "ISO8859-1") + ".xlsx");
        response.setContentType("APPLICATION/msexcel");
        //设置打印对象
        XSSFPrintSetup printSetup = sheet.getPrintSetup();
        //设置打印方向
        printSetup.setLandscape(true);
        // 新建一个Excel的工作空间
        XSSFWorkbook workbook = new XSSFWorkbook();
        // 把模板复制到新建的Excel
        workbook = tpWorkbook;
        // 输出Excel内容，生成Excel文件
        workbook.write(out);
    } catch (final IOException e) {
        e.printStackTrace();
    } finally {
        try {
            // 关闭输出流
            response.flushBuffer();
            if (out != null) {
                out.flush();
                out.close();
            }
        } catch (final IOException e) {
        }
    }
}
```   
```
/**
* 创建表体数据
* @param data - 表体数据
*/
private void setCellData(List<Map<String, Object>> data, String[] colKeys) {
  //创建表头
  createHeader();
  //创建合计项变量
  zq_hs = 0;
  int rowSize = 0;
  // 创建数据
  XSSFRow row = null;
  XSSFCell cell = null;
  int i = 3;//startRow：数据开始行
  if (data != null && data.size() > 0) {
      for (int m = 0; m < data.size(); m++) {
          row = sheet.createRow(i);
          if (i == 3) {//第一行
              rowSize = i;
              setCellValue(data, colKeys, row, cell, m);
          } else {
              if (data.get(m).get("blId") == data.get(m - 1).get("blId")) {//相同字段
                  setCellValue(data, colKeys, row, cell, m);
              } else {//合计行
                  if (rowSize != (i - 1)) {//多于1条数据时合并单元格
                      CellRangeAddress cra = new CellRangeAddress(rowSize, i - 1, 2, 2); // 起始行, 终止行, 起始列, 终止列
                      sheet.addMergedRegion(cra);
                      RegionUtil.setBorderBottom(BorderStyle.THIN, cra, sheet);// 下边框
                      RegionUtil.setBorderRight(BorderStyle.THIN, cra, sheet); // 右边框
                  }
                  //合计行
                  createTailCellWeekly1(i, zq_hs);
                  zq_hs = 0;
                  i = i + 1;
                  rowSize = i;
                  row = sheet.createRow(i);
                  //下一个模块
                  setCellValue(data, colKeys, row, cell, m);
              }
              //项目总合计
              if (m == (data.size() - 1)) {
                  if (rowSize != i) {//多于1条数据时合并单元格
                      CellRangeAddress cra = new CellRangeAddress(rowSize, i, 2, 2); // 起始行, 终止行, 起始列, 终止列
                      sheet.addMergedRegion(cra);
                      RegionUtil.setBorderBottom(BorderStyle.THIN, cra, sheet);// 下边框
                      RegionUtil.setBorderRight(BorderStyle.THIN, cra, sheet); // 右边框
                  }
                  zq_hsZ = zq_hsZ + zq_hs;
                  //合计行
                  i = i + 1;
                  createTailCellWeekly1(i, zq_hs);
              }
          }
          if (data.size() == 1) {
              //项目总计累加
              zq_hsZ = zq_hsZ + zq_hs;/
              i = i + 1;
              //合计行
              createTailCellWeekly1(i, zq_hs);
          }
          i++;//下一行
          //行赋值完成后，对当前行操作
          styleWeekly(row);
      }
  }
  //创建表尾
  createTailCellWeekly2(i - 1, zq_hsZ, zq_mjZ, hd_hsZ, hd_hs_dzwcZ, hd_mjZ, hd_mj_dzwcZ, azf_rwZ, azf_ykgZ, gd_rwZ, gd_ywc_weekZ);
}
```   
```
//创建表头
private void createHeader(String year) {
    // 创建数据
    XSSFCell cell = null;
    XSSFRow row0 = sheet.getRow(0);
    row0.getCell(0).setCellValue(year + "项目计划表");

    XSSFRow row1 = sheet.getRow(1);
    cell = row1.getCell(10);
    cell.setCellValue(year + "项目投资");
    cell = row1.getCell(16);
    cell.setCellValue(year + "项目oo");
}
```   
```
/**
 * @return void
 * @Description //TODO 行赋值
 * @Param [data, colKeys, backupType, row, cell, m]
 */
private void setCellValue(List<Map<String, Object>> data, String[] colKeys XSSFRow row, XSSFCell cell, int m) {
    XSSFCellStyle cellStyleTongJi = tpWorkbook.createCellStyle();
    XSSFFont font = tpWorkbook.createFont();
    font.setFontHeightInPoints((short) 36);// 字体大小
    cellStyleTongJi.setFont(font);
    cellStyleTongJi.setBorderBottom(BorderStyle.THIN); // 下边框
    cellStyleTongJi.setBorderRight(BorderStyle.THIN);// 右边框
    cellStyleTongJi.setBorderLeft(BorderStyle.THIN);//下边框
    cellStyleTongJi.setAlignment(HorizontalAlignment.CENTER_SELECTION);//水平对齐
    cellStyleTongJi.setVerticalAlignment(VerticalAlignment.CENTER);//上下对齐
    cellStyleTongJi.setWrapText(true);//自动换行

    XSSFCellStyle cellStyleTongJi1 = tpWorkbook.createCellStyle();
    cellStyleTongJi1.setFont(font);
    cellStyleTongJi1.setBorderBottom(BorderStyle.THIN); // 下边框
    cellStyleTongJi1.setBorderRight(BorderStyle.THIN);// 右边框
    cellStyleTongJi1.setAlignment(HorizontalAlignment.LEFT);//水平对齐
    cellStyleTongJi1.setVerticalAlignment(VerticalAlignment.CENTER);//上下对齐
    cellStyleTongJi1.setWrapText(true);//自动换行

    int j = 0;
    for (String key : colKeys) {
        Object colValue = data.get(m).get(key);
        cell = row.createCell(j);
        //后2列文字描述左对齐
        if (key.equalsIgnoreCase("col1") || key.equalsIgnoreCase("col2")) {
            cell.setCellStyle(cellStyleTongJi1);
        } else {
            cell.setCellStyle(cellStyleTongJi);
        }
        if (colValue != null) {
            //单元格赋值
            if (key.equalsIgnoreCase("col3")) {
                cell.setCellValue(colValue.toString() + "地区");
            } else if (key.equalsIgnoreCase("col4")) {
                cell.setCellValue(new BigDecimal(colValue.toString()).stripTrailingZeros().toPlainString());//BigDecimal=>number
            } else if (key.equalsIgnoreCase("col5") || key.equalsIgnoreCase("col6") || ) {
                cell.setCellValue(new BigDecimal(colValue.toString()).stripTrailingZeros().toPlainString());//BigDecimal=>number
            } else {
                cell.setCellValue(colValue.toString());
            }
            //计算合计值
            switch (key) {
                case "planMoveCount":
                    zq_hs += Double.parseDouble(String.valueOf(colValue));
                    break;
                case "planMoveArea":
                    zq_mj += Double.parseDouble(String.valueOf(colValue));
                    break;
                case "mYearCount":
                    hd_hs += Double.parseDouble(String.valueOf(colValue));
                    break;
                case "mWeekCount":
                    hd_hs_dzwc += Double.parseDouble(String.valueOf(colValue));
                    break;
            }
        } else {
            cell.setCellValue("--");
        }
        j++;
    }
}
```   
```   
/**
 * @return void
 * @Description //TODO 创建合计项
 * @Param [size, zq_hs]
 */
private void createTailCellWeekly1(int size, double zq_hs) {
    XSSFCellStyle cellStyleTongJi = tpWorkbook.createCellStyle();
    XSSFFont font = tpWorkbook.createFont();
    font.setBold(true);
    font.setFontHeightInPoints((short) 36);// 字体大小
    cellStyleTongJi.setFont(font);
    cellStyleTongJi.setBorderBottom(BorderStyle.THIN); // 下边框
    cellStyleTongJi.setBorderRight(BorderStyle.THIN);// 右边框
    cellStyleTongJi.setBorderLeft(BorderStyle.THIN);//下边框
    cellStyleTongJi.setAlignment(HorizontalAlignment.CENTER_SELECTION);//水平对齐
    cellStyleTongJi.setVerticalAlignment(VerticalAlignment.CENTER);//上下对齐
    cellStyleTongJi.setWrapText(true);//自动换行

    //合计
    XSSFRow rowTail1 = sheet.createRow(size);
    rowTail1.setHeightInPoints((short) 100);
    sheet.addMergedRegion(new CellRangeAddress(size, size, 0, 2));

    XSSFCell cellTail1 = rowTail1.createCell(0);
    cellTail1.setCellValue("合计");
    cellTail1.setCellStyle(cellStyleTongJi);

    XSSFCell cellTail102 = rowTail1.createCell(1);
    cellTail102.setCellValue("");
    cellTail102.setCellStyle(cellStyleTongJi);

    XSSFCell cellTail103 = rowTail1.createCell(2);
    cellTail103.setCellValue(zq_hs);
    cellTail103.setCellStyle(cellStyleTongJi);
}
```
```
/**
 * @return void
 * @Description //TODO 导出完成后格式整理
 * @Param [row]
 */
private void styleWeekly(XSSFRow row){
    for(int k=16;k< 17;k++){
        sheet.autoSizeColumn(k,true);//自适应列宽
        if(row.getCell(k).toString()=="--"){
            row.getCell(k).setCellValue("");
        }
    }
    if(row.getCell(9).toString()!="--"){
        if(row.getCell(3).toString()=="--"){
            row.getCell(3).setCellValue(0);
        }
    }else{
        if(row.getCell(3).toString()=="--"){
            row.getCell(3).setCellValue("--");
        }
    }
}
```
