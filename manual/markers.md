# Log4j 2 API - Markers

## Markers

日志框架的主要目的之一是提供在必要时的调试信息和诊断信息，允许过滤信息，以使其信息量不至于过多或信息是定制的。下面的示例中，应用程序期望exit和其他操作与执行的SQL语句分开地记录，并且希望还能够区分query与update语句。如下所示：

```java
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.MarkerManager;
import java.util.Map;

public class MyApp {

    private Logger logger = LogManager.getLogger(MyApp.class.getName());
    private static final Marker SQL_MARKER = MarkerManager.getMarker("SQL");
    private static final Marker UPDATE_MARKER = MarkerManager.getMarker("SQL_UPDATE").setParents(SQL_MARKER);
    private static final Marker QUERY_MARKER = MarkerManager.getMarker("SQL_QUERY").setParents(SQL_MARKER);

    public String doQuery(String table) {
        logger.entry(param);

        logger.debug(QUERY_MARKER, "SELECT * FROM {}", table);

        return logger.exit();
    }

    public String doUpdate(String table, Map<String, String> params) {
        logger.entry(param);

        if (logger.isDebugEnabled()) {
          logger.debug(UPDATE_MARKER, "UPDATE {} SET {}", table, formatCols());
        }
        return logger.exit();
    }

    private String formatCols(Map<String, String> cols) {
        StringBuilder sb = new StringBuilder();
        boolean first = true;
        for (Map.Entry<String, String> entry : cols.entrySet()) {
            if (!first) {
                sb.append(", ");
            }
            sb.append(entry.getKey()).append("=").append(entry.getValue());
            first = false;
        }
        return sb.toString();
    }
}
```

在上面的示例中，可以添加MarkerFilters来达到这些目录，如仅仅允许记录SQL update操作的日志信息。

使用Marker时，必须考虑这些重要规则：

1. Marker必须是唯一的。它们是通过名字注册的，所以非必需情况下应注意确保您的应用程序中使用的Marker与其他的Marker区别开来。
2. Parent Marker可以动态add或remove。然而，这个做法的代价是昂贵的（同步操作）。相反，建议在首次获取Marker时进行parents设置，如上面的示例做法。具体来说，set方法是批量处理的，而add和remover操作是逐个处理的。
3. 处理具有多个祖先的Marker比没有parent的Marker代价要大得多。例如，在一次性能测试中，处理一个Marker是否匹配其祖父节点比处理Marker本身花费的时间大3倍。即使如此，处理Marker与解析调用者类名或行号相比是快很多的。
