# Log4j 2 API - Event Logging

## 事件记录（Event Logging）

EventLogger类提供了一种简单机制，用于记录应用程序中发生的事件。虽然EventLogger在初始化那些应由审计日志系统处理的事件这个功能上是有用的，但它本身并不实现审计日志系统的任何功能，例如保证传递性。

在典型的Web应用程序中使用EventLogger时，建议把整个生命周期请求的（例如用户的ID，用户的IP地址，产品名称等）相关的数据放置于ThreadContext Map中。这点在servlet filter中很容易实现，其中ThreadContext Map也可以在请求结束时被清除。当需要记录事件时，应创建StructuredDataMessage。然后调用EventLogger.logEvent(StructuredDataMessage msg)。

```java
import org.apache.logging.log4j.ThreadContext;
import org.apache.commons.lang.time.DateUtils;

import javax.servlet.Filter;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.FilterChain;
import javax.servlet.http.HttpSession;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.TimeZone;

public class RequestFilter implements Filter {
    private FilterConfig filterConfig;
    private static String TZ_NAME = "timezoneOffset";

    public void init(FilterConfig filterConfig) throws ServletException {
        this.filterConfig = filterConfig;
    }

    /**
     * Sample filter that populates the MDC on every request.
     */
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest)servletRequest;
        HttpServletResponse response = (HttpServletResponse)servletResponse;
        ThreadContext.put("ipAddress", request.getRemoteAddr());
        HttpSession session = request.getSession(false);
        TimeZone timeZone = null;
        if (session != null) {
            // Something should set this after authentication completes
            String loginId = (String)session.getAttribute("LoginId");
            if (loginId != null) {
                ThreadContext.put("loginId", loginId);
            }
            // This assumes there is some javascript on the user's page to create the cookie.
            if (session.getAttribute(TZ_NAME) == null) {
                if (request.getCookies() != null) {
                    for (Cookie cookie : request.getCookies()) {
                        if (TZ_NAME.equals(cookie.getName())) {
                            int tzOffsetMinutes = Integer.parseInt(cookie.getValue());
                            timeZone = TimeZone.getTimeZone("GMT");
                            timeZone.setRawOffset((int)(tzOffsetMinutes * DateUtils.MILLIS_PER_MINUTE));
                            request.getSession().setAttribute(TZ_NAME, tzOffsetMinutes);
                            cookie.setMaxAge(0);
                            response.addCookie(cookie);
                        }
                    }
                }
            }
        }
        ThreadContext.put("hostname", servletRequest.getServerName());
        ThreadContext.put("productName", filterConfig.getInitParameter("ProductName"));
        Threadcontext.put("locale", servletRequest.getLocale().getDisplayName());
        if (timeZone == null) {
            timeZone = TimeZone.getDefault();
        }
        ThreadContext.put("timezone", timeZone.getDisplayName());
        filterChain.doFilter(servletRequest, servletResponse);
        ThreadContext.clear();
    }

    public void destroy() {
    }
}
```

使用EventLogger的示例类。

```java
import org.apache.logging.log4j.StructuredDataMessage;
import org.apache.logging.log4j.EventLogger;

import java.util.Date;
import java.util.UUID;

public class MyApp {

    public String doFundsTransfer(Account toAccount, Account fromAccount, long amount) {
        toAccount.deposit(amount);
        fromAccount.withdraw(amount);
        String confirm = UUID.randomUUID().toString();
        StructuredDataMessage msg = new StructuredDataMessage(confirm, null, "transfer");
        msg.put("toAccount", toAccount);
        msg.put("fromAccount", fromAccount);
        msg.put("amount", amount);
        EventLogger.logEvent(msg);
        return confirm;
    }
}
```

EventLogger类使用名为“EventLogger”的Logger。EventLogger使用级别OFF作为默认值，默认无法记录。
