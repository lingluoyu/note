### 引入itext

```
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itext7-core</artifactId>
    <version>7.1.13</version>
    <type>pom</type>
</dependency>
```
### 使用itext
#### HelloWorld
```
PdfWriter writer = new PdfWriter(DEST);
PdfDocument pdf = new PdfDocument(writer);
Document document = new Document(pdf);

document.add(new Paragraph("Hello World!"));
document.close();
```
**生成pdf如下**

![itext_01](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203140205.png)

#### 更改字体

```java
PdfWriter writer = new PdfWriter(DEST);
PdfDocument pdf = new PdfDocument(writer);
Document document = new Document(pdf);

//选择字体
PdfFont font = PdfFontFactory.createFont("STSong-Light", "UniGB-UCS2-H", false);

document.add(new Paragraph("字体展示：").setFont(font));

List list = new List()
        .setSymbolIndent(12)
        .setListSymbol("\u2022")
        .setFont(font);
// Add ListItem objects
list.add(new ListItem("永远不要放弃/Never gonna give you up"))
        .add(new ListItem("Never gonna let you down"))
        .add(new ListItem("Never gonna run around and desert you"))
        .add(new ListItem("Never gonna make you cry"))
        .add(new ListItem("Never gonna say goodbye"))
        .add(new ListItem("Never gonna tell a lie and hurt you"));
// Add the list
document.add(list);
document.close();
```

**生成pdf如下**

![itext_02](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203142903.png)

#### 插入图片

```java
PdfWriter writer = new PdfWriter(DEST);
PdfDocument pdf = new PdfDocument(writer);
Document document = new Document(pdf);

Image image = new Image(ImageDataFactory.create(TEST));

document.add(new Paragraph("Insert Image ").add(image));
document.close();
```

**生成pdf如下**

![itext_03](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203145806.png)

#### 生成表格

```java
PdfWriter writer = new PdfWriter(DEST);
PdfDocument pdf = new PdfDocument(writer);
Document document = new Document(pdf, PageSize.A4.rotate());
document.setMargins(20, 20, 20, 20);

PdfFont font = PdfFontFactory.createFont(StandardFonts.HELVETICA);
PdfFont bold = PdfFontFactory.createFont(StandardFonts.HELVETICA_BOLD);
Table table = new Table(UnitValue.createPercentArray(new float[]{4, 1, 3, 4, 3, 3, 3, 3, 1}))
        .useAllAvailableWidth();
BufferedReader br = new BufferedReader(new FileReader(DATA));
String line = br.readLine();
process(table, line, bold, true);
while ((line = br.readLine()) != null) {
    process(table, line, font, false);
}
br.close();
document.add(table);
document.close();


public static void process(Table table, String line, PdfFont font, boolean isHeader) {
    StringTokenizer tokenizer = new StringTokenizer(line, ";");
    while (tokenizer.hasMoreTokens()) {
        if (isHeader) {
            table.addHeaderCell(new Cell().add(new Paragraph(tokenizer.nextToken()).setFont(font)));
        } else {
            table.addCell(new Cell().add(new Paragraph(tokenizer.nextToken()).setFont(font)));
        }
    }
}
```

**生成pdf如下**

![itext_04](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203151633.png)



### 根据模板填入数据

#### 创建模板

**手动编辑模板**

1. 使用word创建模板页面“模板.docx”

   ![itext_05](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203170527.png)

2. 另存为pdf格式”模板.pdf“

   ![itext_06](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203170610.png)

3. 打开Adobe Acrobat pro

4. 点击“准备表单”按钮，选择“模板.pdf”

5. 可选择修改表单域名称

   ![itext_07](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203170631.png)

6. 另存为”模板V2.pdf“

**代码生成模板**

```java
public class PdfUtils {

    public static void createTemplatePdf(String dest) throws IOException {
        //初始化pdf文档
        PdfDocument pdf = new PdfDocument(new PdfWriter(dest));
        pdf.setDefaultPageSize(PageSize.A4);

        //初始化文档
        Document document = new Document(pdf);
        //中文显示
        PdfFont font = PdfFontFactory.createFont("STSong-Light", "UniGB-UCS2-H", false);
        document.setFont(font);

        addForm(document);

        //关闭文档
        document.close();
    }

    private static PdfAcroForm addForm(Document doc) {
        //模板标题
        Paragraph title = new Paragraph("个人信息表")
                .setTextAlignment(TextAlignment.CENTER)
                .setFontSize(16);
        doc.add(title);
        //模板内容
        doc.add(new Paragraph("姓名").setFontSize(12));
        doc.add(new Paragraph("性别").setFontSize(12));
        doc.add(new Paragraph("民族").setFontSize(12));
        doc.add(new Paragraph("身份证号").setFontSize(12));
        doc.add(new Paragraph("家庭住址").setFontSize(12));
        doc.add(new Paragraph("个人介绍").setFontSize(12));

        PdfDocument pdfDocument = doc.getPdfDocument();

        //创建模板
        PdfAcroForm form = PdfAcroForm.getAcroForm(pdfDocument, true);

        //表单域
        PdfTextFormField nameField = PdfTextFormField.createText(pdfDocument,
                new Rectangle(154, 117, 73, 15), "name", "");
        form.addField(nameField);

        PdfTextFormField genderField = PdfTextFormField.createText(pdfDocument,
                new Rectangle(300, 117, 73, 15), "gender", "");
        form.addField(genderField);

        PdfTextFormField nationField = PdfTextFormField.createText(pdfDocument,
                new Rectangle(430, 117, 73, 15), "nation", "");
        form.addField(nationField);

        PdfTextFormField IDNumberField = PdfTextFormField.createText(pdfDocument,
                new Rectangle(154, 135, 347, 15), "IDNumber", "");
        form.addField(IDNumberField);

        PdfTextFormField addressField = PdfTextFormField.createText(pdfDocument,
                new Rectangle(154, 149, 347, 15), "address", "");
        form.addField(addressField);

        PdfTextFormField introductionField = PdfTextFormField.createText(pdfDocument,
                new Rectangle(154, 166, 347, 15), "introduction", "");
        form.addField(introductionField);

        return form;
    }
}
```

#### pdf生成代码

```java
public static final void replaceTextFieldPdf(String templatePdfPath, String destPdfPath,
                                                 Map<String, String> params) {
        PdfDocument pdf = null;
        try {
            // 判断文件是否存在
            File file = new File(destPdfPath);
            if (!file.getParentFile().exists()) {
                file.getParentFile().mkdirs();
            }
            // 有参数才替换
            if (params != null && !params.isEmpty()) {
                pdf = new PdfDocument(new PdfReader(templatePdfPath), new PdfWriter(destPdfPath));
                PdfAcroForm form = PdfAcroForm.getAcroForm(pdf, true);
                Map<String, PdfFormField> fields = form.getFormFields(); // 获取所有的表单域
                for (String param : params.keySet()) {
                    PdfFormField formField = fields.get(param); // 获取某个表单域
                    if (formField != null) {
                        formField.setFont(getImportFont("方正粗黑宋简体.ttf")).setValue(params.get(param)); // 替换值
//                formField.setFont(getDefaultFont()).setValue(params.get(param)); // 替换值
                    }
                }
                form.flattenFields();// 锁定表单，不让修改
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (pdf != null) {
                pdf.close();
            }
        }
    }
```

**生成文件如图**

![itext_08](https://raw.githubusercontent.com/lingluoyu/image/master/img/20201203170640.png)



















