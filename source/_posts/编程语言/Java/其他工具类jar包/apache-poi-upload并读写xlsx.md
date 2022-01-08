---
title: apache-poi-upload并读写xlsx等文件
date: 2021-04-07 08:46:02
tags:
---

## spring boot上传CSV文件

### pom文件

```pom
<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-csv</artifactId>
	<version>1.8</version>
</dependency>
```

## Spring boot上传xlsx文件

### pom文件

```pom
<dependency>
	<groupId>org.apache.poi</groupId>
	<artifactId>poi-ooxml</artifactId>
	<version>4.1.2</version>
</dependency>
```

### 读写逻辑类

- 如果想从指定line读取， Iterator<Row> rows可以循环Line次，把iterator指向Line

```java
import org.apache.poi.ss.usermodel.Cell;
import org.apache.poi.ss.usermodel.Row;
import org.apache.poi.ss.usermodel.Sheet;
import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.web.multipart.MultipartFile;

public class ExcelHelper {
  public static String TYPE = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";
  static String[] HEADERs = { "Id", "Title", "Description", "Published" };
  static String SHEET = "Tutorials";

  public static boolean hasExcelFormat(MultipartFile file) {
    if (!TYPE.equals(file.getContentType())) {
      return false;
    }

    return true;
  }

  public static List<Tutorial> excelToTutorials(InputStream is) {
    try {
      Workbook workbook = new XSSFWorkbook(is);

      Sheet sheet = workbook.getSheet(SHEET);
      Iterator<Row> rows = sheet.iterator();

      List<Tutorial> tutorials = new ArrayList<Tutorial>();

      int rowNumber = 0;
      while (rows.hasNext()) {
        Row currentRow = rows.next();

        // skip header
        if (rowNumber == 0) {
          rowNumber++;
          continue;
        }

        Iterator<Cell> cellsInRow = currentRow.iterator();

        Tutorial tutorial = new Tutorial();

        int cellIdx = 0;
        while (cellsInRow.hasNext()) {
          Cell currentCell = cellsInRow.next();

          //最好用CellType进行判断
          if(MytempCell.getCellType() == CellType.CELL_TYPE_NUMERIC)
           your code ...
          else if(MytempCell.getCellType() == CellType.CELL_TYPE_STRING)
           your code ...
            
          //类型转换，防止手机号读成其他格式
          if(cell.getCellType() != Cell.CELL_TYPE_STRING){
              cell.setCellType(Cell.CELL_TYPE_STRING);
          }
           
          switch (cellIdx) {
            case 0:
              tutorial.setId((long) currentCell.getNumericCellValue());
              break;

            case 1:
              tutorial.setTitle(currentCell.getStringCellValue());
              break;

            default:
              break;
          }

          cellIdx++;
        }

        tutorials.add(tutorial);
      }

      workbook.close();

      return tutorials;
    } catch (IOException e) {
      throw new RuntimeException("fail to parse Excel file: " + e.getMessage());
    }
  }
}
```

## MVC架构

### service类

```java
@Service
public class ExcelService {
  @Autowired
  TutorialRepository repository;

  public void save(MultipartFile file) {
    try {
      List<Tutorial> tutorials = ExcelHelper.excelToTutorials(file.getInputStream());
    } catch (IOException e) {
      throw new RuntimeException("fail to store excel data: " + e.getMessage());
    }
  }
}
```

### 上传xlsx文件

```java
@Controller
@RequestMapping("/api/excel")
public class ExcelController {
  @Autowired
  ExcelService fileService;

  @PostMapping("/upload")
  public String uploadFile(@RequestParam("file") MultipartFile file) {
    String message = "";

    if (ExcelHelper.hasExcelFormat(file)) {
      try {
        fileService.save(file);

        message = "Uploaded the file successfully: " + file.getOriginalFilename();
        return message
      } catch (Exception e) {
        message = "Could not upload the file: " + file.getOriginalFilename() + "!";
        return message
      }
    }

    message = "Please upload an excel file!";
    return message;
  }
}
```

