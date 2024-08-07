### 日期与时间的处理

```java
public void dealDate() {
    Date date = new Date();
    //Fri Feb 12 09:08:23 GMT+08:00 2021
    System.out.println(date);

    //当前时间戳（不包含当前时区信息）
    Instant instant = Instant.now();
    //2021-02-12T03:58:06.792Z
    System.out.println(instant);

    //当前时间戳
    LocalDateTime localDateTime = LocalDateTime.now();
    //2021-02-12T11:58:06.846
    System.out.println(localDateTime);

    //当前日期
    LocalDate today = LocalDate.now();
    //2021-02-12
    System.out.println(today);

    //分别获取年月日
    int year = today.getYear();
    int month = today.getMonthValue();
    int dayOfMonth = today.getDayOfMonth();
    //Year:2021  Month:2  day:12 
    System.out.printf("Year:%d  Month:%d  day:%d%n", year, month, dayOfMonth);

    //创建特定的日期
    LocalDate specialDate = LocalDate.of(2020, 12, 14);
    //The special date = 2020-12-14
    System.out.println("The special date = " + specialDate);

    //比较时间是否相等
    LocalDate date1 = LocalDate.of(2021, 2, 12);
    boolean equalFlag = LocalDate.now().equals(date1);
    //Is time equal or not ? -- true
    System.out.println("Is time equal or not ? -- " + equalFlag);

    //处理周期性的日期
    LocalDate single2018 = LocalDate.of(2018, 11, 11);
    MonthDay singlesDay = MonthDay.of(single2018.getMonth(), single2018.getDayOfMonth());
    MonthDay currentMonthDay = MonthDay.from(LocalDate.now());
    boolean equalFlag1 = singlesDay.equals(currentMonthDay);
    //Is it 11.11 today? -- false
    System.out.println("Is it 11.11 today? -- " + equalFlag1);

    //当前时分秒
    LocalTime nowTime = LocalTime.now();
    //10:19:38.760
    System.out.println(nowTime);

    //三小时后时间
    LocalTime timePlus = nowTime.plusHours(3);
    //The time after 3 hours = 13:22:11.569
    System.out.println("The time after three hours = " + timePlus);

    //一周后的日期
    LocalDate nextWeek = LocalDate.now().plus(1, ChronoUnit.WEEKS);
    //The Date after 1 week = 2021-02-19
    System.out.println("The Date after 1 week = " + nextWeek);

    //一年前的日期
    LocalDate previousYear = LocalDate.now().minus(1, ChronoUnit.YEARS);
    //The Date before 1 year = 2020-02-12
    System.out.println("The Date before 1 year = " + previousYear);

    //时钟类
    Clock clock = Clock.systemUTC();
    //SystemClock[Z]
    System.out.println(clock);
    Clock defaultClock = Clock.systemDefaultZone();
    //SystemClock[GMT+08:00]
    System.out.println(defaultClock);

    //判断日期早晚
    boolean afterFlag = specialDate.isAfter(LocalDate.now());
    //2020-12-14 is after now ? -- false
    System.out.println(specialDate + " is after now ? -- " + afterFlag);
    boolean beforeFlag = specialDate.isBefore(LocalDate.now());
    //2020-12-14 is before now ? -- true
    System.out.println(specialDate + " is before now ? -- " + beforeFlag);

    //处理时区
    ZoneId america = ZoneId.of("America/New_York");
    LocalDateTime dateTime = LocalDateTime.now();
    ZonedDateTime zonedDateTime = ZonedDateTime.of(dateTime, america);
    //当前时区的时间与指定时区的时差： 2021-02-12T11:13:37.487-05:00[America/New_York]
    System.out.println("当前时区的时间与指定时区的时差： " + zonedDateTime);

    //YearMonth处理特定日期
    YearMonth currentYearMonth = YearMonth.now();
    //2021-02 include 28 days.
    System.out.printf("%s include %d days.%n", currentYearMonth, currentYearMonth.lengthOfMonth());

    //闰年
    boolean leapYear = LocalDate.now().isLeapYear();
    //This year is leap year? -- false
    System.out.println("This year is leap year? -- " + leapYear);

    //计算两个日期间的间隔
    Period betweenTime = Period.between(specialDate, LocalDate.now());
    //Interval between two dates are 1 mouths 29 days.
    System.out.printf("Interval between two dates are %d mouths %d days.%n", betweenTime.getMonths(),
        betweenTime.getDays());
    long betweenDays = ChronoUnit.DAYS.between(specialDate, LocalDate.now());
    //Interval between two dates are 60 days.
    System.out.printf("Interval between two dates are %d days.%n", betweenDays);

    //日期和时间中包含时差
    ZoneOffset offset = ZoneOffset.of("+05:00");
    OffsetDateTime offsetDateTime = OffsetDateTime.of(LocalDateTime.now(), offset);
    //dateTime with offset is : 2021-02-12T11:46:26.081+05:00
    System.out.println("dateTime with offset is : " + offsetDateTime);

    //解析格式化日期
    String dateStr = "20090901";
    LocalDate formatDate = LocalDate.parse(dateStr, DateTimeFormatter.BASIC_ISO_DATE);
    //20090901 parse to date 2009-09-01
    System.out.printf("%s parse to date %s%n", dateStr, formatDate);
}
```

