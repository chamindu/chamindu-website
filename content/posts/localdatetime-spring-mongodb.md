---
title: "Persisting LocalDateTime Values in Mongodb With Spring"
date: 2019-08-08T20:09:03+05:30
draft: false
tags:
  - Mongodb
  - Spring
  - Java
  - Date time
categories:
  - "Software Engineering"
images:
  - images/logo-300x300.png
---

*The sample application used for this article is available in [github](https://github.com/chamindu/mongo-time).It expects a local MongoDB instance and it will create a database called mongotime.*

If you are using MongoDB as your storage backend in a Spring application you are most likely
using [spring-data-mongodb](https://spring.io/projects/spring-data-mongodb). MongoDB uses BSON
as the underlying storage and wire format. BSON only has one type for date time storage and by
[definition](http://bsonspec.org/spec.html) it only allows UTC. If your models
contain `LocalDateTime` attributes they will be converted back and forth from UTC using your current system time zone. 

You can see this in action in the sample application provided. The fragment below lists the `Event` entity that uses a 'LocalDateTime' field.

[Event.java](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/Event.java)
{{< highlight java >}}
public class Event {
    private LocalDateTime eventTime;

    public LocalDateTime getEventTime() {
        return this.eventTime;
    }

    public void setEventTime(LocalDateTime eventTime) {
        this.eventTime = eventTime;
    }
}
{{< /highlight >}}

The application creates a new `Event` with an event time of `2019-08-01T09:30:00`.

[MongotimeApplication.java](https://github.com/chamindu/mongo-time/blob/ef502efb4f082a4c8991b097b3ac307fa6b0549b/src/main/java/dev/chamindu/mongotime/MongotimeApplication.java#L31)
{{< highlight java >}}
private void createEvent() {
    System.out.println("Deleting old event records.");
    this.eventRepository.deleteAll();

    LocalDateTime dateTime = LocalDateTime.of(2019, 8, 1, 9, 30);
    Event event = new Event();
    event.setEventTime(dateTime);

    this.eventRepository.save(event);
}
{{< /highlight >}}

Lets see whats in the event collection in MongoDB.

{{< highlight json >}}
{
    "_id" : ObjectId("5d4c621eaf6794102362ee6c"),
    "eventTime" : ISODate("2019-08-01T04:00:00.000Z"),
    "_class" : "dev.chamindu.mongotime.Event"
}
{{< /highlight >}}

Since my computer is on GMT+5:30 the `LocalDateTime` value was translated to `2019-08-01T04:00:00.000Z`. While this may work for some cases it falls short 
when you want to store the local date time as is. After all the java `LocalDateTime` type
is meant to represent a date and time without timezone information.

I came across such a scenario in one of the projects I worked on. We capture events when customers enter and exit a retail store and publish them to a cloud based backend for analysis. There are a 
large number of stores located in different timezones. When we do the analytics we want to compare
store traffic side by side in local time of each location since store operating hours are defined in local time.

## Storing Date and Time as Strings

One solution for the above mentioned problem is to store the datetime as a string. But you have to 
be careful about the format that you use. Its best to use ISO formated dates because the date parts 
are ordered from the most significant to the least significant. Due to the above mentioned property
the lexical ordering and temporal ordering is the same and you can use a string rage query to filter data in a date range.

Although its a good idea to store dates as strings in MongoDb its not a great idea to use String fields to represent dates in your Java models. You field becomes weakly typed and you lose the ability to do date arithmetic.

## Developing a Custom Temporal Type

To overcome the issues discussed above we can use a custom type. The basic idea is to wrap an instance of `LocalDateTime` inside our type. This will make sure Spring Data does not apply
the default conversion to BSON date. Then we get the assistance of the 
Spring type converter system to convert it to `String` for storage and vice versa. 


[MongoLocalDateTime.java](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/MongoLocalDateTime.java)
{{< highlight java >}}
public class MongoLocalDateTime implements Comparable<MongoLocalDateTime> {

    private LocalDateTime localDateTime;

    private MongoLocalDateTime(LocalDateTime localDateTime) {
        this.localDateTime = localDateTime;
    }

    public static MongoLocalDateTime of(LocalDateTime localDateTime) {
        return new MongoLocalDateTime(localDateTime);
    }

    public static MongoLocalDateTime of(int year, int month, int dayOfMonth, int hour, int minute, int second) {
        return new MongoLocalDateTime(LocalDateTime.of(year, month, dayOfMonth, hour, minute, second));
    }

    public int getYear() {
        return this.localDateTime.getYear();
    }

    public Month getMonth() {
        return this.localDateTime.getMonth();
    }
    
    public int getDayOfMonth() {
        return this.localDateTime.getDayOfMonth();
    }

    public int getHour() {
        return this.localDateTime.getHour();
    }

    public int getMinute() {
        return this.localDateTime.getMinute();
    }

    public int getSecond() {
        return this.localDateTime.getSecond();
    }

    public LocalDateTime toLocalDateTime() {
        return this.localDateTime;
    }

    public MongoLocalDateTime plusDays(long days) {
        return new MongoLocalDateTime(this.localDateTime.plusDays(days));
    }

    @Override
    public int compareTo(MongoLocalDateTime other) {
        return this.localDateTime.compareTo(other.localDateTime);
    }

    public String toString() {
        return  this.localDateTime.toString();
    }
}

{{< /highlight >}}

The `MongoLocalDateTime` type listed above simply delegates most of its functionality to the 
underlying `LocalDateTime` field. One thing to remember is that types in `java.time` 
package are immutable and you should design your type to be immutable as well.

## The Type Converters

Spring type converters are used to convert from an instance of a given source type to a target type. Spring uses type converters in many places including MVC model binding, bean configuration etc. Spring Data MongoDB also uses type converters to convert back and forth from BSON types to Java types. For our `MongoLocalDateTime` type we will need two converters one to convert from `MongoLocalDateTime` to `String` when we save data to the DB and another one to convert `String` to `MongoLocalDateTime` when we read data.

[MongoLocalDateTimeToStringConverter.java](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/MongoLocalDateTimeToStringConverter.java)
{{< highlight java >}}
public class MongoLocalDateTimeToStringConverter implements Converter<MongoLocalDateTime, String> {

    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSS");

    @Override
    public String convert(MongoLocalDateTime source) {
        return formatter.format(source.toLocalDateTime());
    }
}
{{< /highlight >}}


[StringToMongoLocalDateTimeConverter](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/StringToMongoLocalDateTimeConverter.java)
{{< highlight java "linenos=table" >}}
public class StringToMongoLocalDateTimeConverter implements Converter<String, MongoLocalDateTime> {

    private static final TypeDescriptor SOURCE = TypeDescriptor.valueOf(String.class);
    private static final TypeDescriptor TARGET = TypeDescriptor.valueOf(MongoLocalDateTime.class);

    @Override
    public MongoLocalDateTime convert(String source) {
        try {
            return MongoLocalDateTime.of(LocalDateTime.parse(source));
        } catch (DateTimeParseException ex) {
            throw new ConversionFailedException(SOURCE, TARGET, source, ex);
        }
    }
}
{{< /highlight >}}

Finally we need to instruct spring data to use the type converters using a custom mongo configuration.

[MongoConfiguration.java](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/MongoConfiguration.java)
{{< highlight java >}}
@Configuration
public class MongoConfiguration extends AbstractMongoConfiguration {

    @Value("${spring.data.mongodb.database:mongotime}")
    String database;

    @Value("${spring.data.mongodb.host:localhost}:${spring.data.mongodb.port:27017}")
    String host;

    @Override
    protected String getDatabaseName() {
        return database;
    }

    @Bean
    @Override
    public MongoCustomConversions customConversions() {
        List<Converter<?, ?>> converterList = new ArrayList<>();
        converterList.add(new MongoLocalDateTimeToStringConverter());
        converterList.add(new StringToMongoLocalDateTimeConverter());

        return new MongoCustomConversions(converterList);
    }

    @Override
    public MongoClient mongoClient() {
        return new MongoClient(host);
    }
}
{{< /highlight >}}

## Putting it All Together
Now we have all the plumbing required to use our custom date time type. Lets define an Entity and 
a repository together with some code to use them.

[LocalTimeEvent.java](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/LocalTimeEvent.java)
{{< highlight java >}}
public class LocalTimeEvent {
    private MongoLocalDateTime eventTime;

    public MongoLocalDateTime getEventTime() {
        return this.eventTime;
    }

    public void setEventTime(MongoLocalDateTime eventTime) {
        this.eventTime = eventTime;
    }
}
{{< /highlight >}}


[LocalTimeEventRepository.java](https://github.com/chamindu/mongo-time/blob/master/src/main/java/dev/chamindu/mongotime/LocalTimeEventRepository.java)
{{< highlight java >}}
public interface LocalTimeEventRepository extends MongoRepository<LocalTimeEvent, String> {
    
    @Query("{'eventTime':{$gte:'?0', $lte:'?1'}}")
    List<LocalTimeEvent> findByEventTimeRange(MongoLocalDateTime startTime, MongoLocalDateTime end);
}
{{< /highlight >}}


The `LocalTimeEventRepository.findByEventTimeRange` method demonstrates that we can also use our type in custom query methods as well.

The below code segment from `MongotimeApplication` invokes our repository.

[MongotimeApplication](https://github.com/chamindu/mongo-time/blob/9c9f13c6042811d1157ff3111a3f8818af39c1b0/src/main/java/dev/chamindu/mongotime/MongotimeApplication.java#L42)
{{< highlight java >}}
private void createLocalTimeEvents() {
    System.out.println("Deleting old LocalTimeEvent documents and creating 10 new documents.");
    this.localTimeEventRepository.deleteAll();

    for (int day = 1; day < 11; day++) {
        MongoLocalDateTime dateTime = MongoLocalDateTime.of(2019, 8, day, 8, 0, 0);
        LocalTimeEvent event = new LocalTimeEvent();
        event.setEventTime(dateTime);
        this.localTimeEventRepository.save(event);
    }
}

private void queryByDateRange() {
    System.out.println("Querying records in date range 2019-08-05 to 2019-08-08.");

    MongoLocalDateTime start = MongoLocalDateTime.of(2019, 8, 5, 0, 0, 0);
    MongoLocalDateTime end = MongoLocalDateTime.of(2019, 8, 8, 0, 0, 0);

    List<LocalTimeEvent> result = this.localTimeEventRepository.findByEventTimeRange(start, end);

    for(LocalTimeEvent event: result) {
        System.out.println(event.getEventTime());
    }
}
{{< /highlight >}}

## Conclusion
Using the above method you can get around the problem of MongoDB only supporting UTC date time values.
Another neat trick you can do is that you can define new temporal types that has lower precision. In my real world project I used a `MongoLocalDateHour` type that represents a date and a hour in local time.

Please share your thoughts in comments. 
