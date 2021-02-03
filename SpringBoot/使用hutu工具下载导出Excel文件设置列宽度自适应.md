```java
    /**
     * 导出excel
     */
    public static void downloadExcel(List<Map<String, Object>> list, HttpServletResponse response) throws IOException {
        String tempPath = SYS_TEM_DIR + IdUtil.fastSimpleUUID() + ".xlsx";
        File file = new File(tempPath);
        BigExcelWriter writer = ExcelUtil.getBigWriter(file);
        // 一次性写出内容，使用默认样式，强制输出标题
        writer.write(list, true);
        SXSSFSheet sheet = (SXSSFSheet)writer.getSheet();
        //上面需要强转SXSSFSheet  不然没有trackAllColumnsForAutoSizing方法
        sheet.trackAllColumnsForAutoSizing();
        for (int i = 0; i < writer.getColumnCount(); i++) {
            sheet.autoSizeColumn(i);
            //手动调整列宽，解决中文不能自适应问题
            //单元格单行最长支持255*256宽度（每个单元格样式已经设置自动换行，超出即换行）
            //设置最低列宽度，列宽约六个中文字符
            // sheet.getColumnWidth(i)*17/10 为经验值
            int width = Math.max(15 * 256, Math.min(255 * 256, sheet.getColumnWidth(i)*17/10));
            sheet.setColumnWidth(i, width);
        }
        //response为HttpServletResponse对象
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8");
        //test.xls是弹出下载对话框的文件名，不能为中文，中文请自行编码
        response.setHeader("Content-Disposition", "attachment;filename=file.xlsx");
        ServletOutputStream out = response.getOutputStream();
        // 终止后删除临时文件
        file.deleteOnExit();
        writer.flush(out, true);
        //此处记得关闭输出Servlet流
        IoUtil.close(out);
    }

```

> 方法：将数据导入到Excel中之后，读取输出流BigExcelWriter的sheet，对每个列进行宽度自适应调整，大小为Math.max(15 * 256, Math.min(255 * 256, sheet.getColumnWidth(i)*17/10))， 确保列的最大宽度与最小宽度，正常值为列宽的1.7倍，为经验值，其实1.5倍—1.7倍大小都是可以的。