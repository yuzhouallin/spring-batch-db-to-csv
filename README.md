Spring Batch- Read From MySQL database & write to CSV file
Created on: August 4, 2014 | Last updated on: March 12, 2017  websystiqueadmin

 
In this post we will learn about how to use Spring Batch to read from MySQL database using JdbcCursorItemReader and write to a Flat file using FlatFileItemWriter. We will also witness the usage of JobExecutionListener and itemProcessor. Let’s get started.

Other interesting posts you may like
Spring Boot+AngularJS+Spring Data+Hibernate+MySQL CRUD App
Spring Boot REST API Tutorial
Spring Boot WAR deployment example
Spring Boot Introduction + Hello World Example
Secure Spring REST API using OAuth2
AngularJS+Spring Security using Basic Authentication
Secure Spring REST API using Basic Authentication
Spring 4 Caching Annotations Tutorial
Spring 4 Cache Tutorial with EhCache
Spring 4 MVC+JPA2+Hibernate Many-to-many Example
Spring 4 Email Template Library Example
Spring 4 Email With Attachment Tutorial
Spring 4 Email Integration Tutorial
Spring MVC 4+JMS+ActiveMQ Integration Example
Spring 4+JMS+ActiveMQ @JmsLister @EnableJms Example
Spring 4+JMS+ActiveMQ Integration Example
Spring MVC 4+Spring Security 4 + Hibernate Integration Example
Spring MVC 4+Apache Tiles 3 Integration Example
Spring MVC 4+AngularJS Example
Spring MVC 4+AngularJS Routing with UI-Router Example
Spring MVC 4 HelloWorld – Annotation/JavaConfig Example
Spring MVC 4+Hibernate 4+MySQL+Maven integration example
Spring 4 Hello World Example
Spring Security 4 Hello World Annotation+XML Example
Hibernate MySQL Maven Hello World Example (Annotation)
TestNG Hello World Example
JAXB2 Helloworld Example
Spring Batch- Read a CSV file and write to an XML file
Following technologies being used:

Spring Batch 3.0.1.RELEASE
Spring core 4.0.6.RELEASE
Spring jdbc 4.0.6.RELEASE
MySQL Server 5.6
Joda Time 2.3
JDK 1.6
Eclipse JUNO Service Release 2
Let’s begin.

Step 1: Create project directory structure

Following will be the final project structure:

SpringBatchDatabaseToCsv_img1

We will be reading MySQL database and write to a flat file (project/csv/examResult.txt).

Step 2: Create Database Table and populate it with sample data

Create a fairly simple table in MySQL database which maps to our domain model(and sufficient for this example).

create table EXAM_RESULT (
   student_name VARCHAR(30) NOT NULL,
   dob DATE NOT NULL,
   percentage  double NOT NULL
);
 
insert into exam_result(student_name,dob,percentage) 
value('Brian Burlet','1985-02-01',76),('Rita Paul','1993-02-01',92),('Han Yenn','1965-02-01',83),('Peter Pan','1987-02-03',62);

 
Please visit MySQL installation on Local PC in case you are finding difficulties in setting up MySQL locally.


 
Now let’s add all contents mentioned in project structure in step 1.

Step 3: Update pom.xml to include required dependencies

Following is the updated minimalistic pom.xml

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>com.websystique.springbatch</groupId>
  <artifactId>SpringBatchDatabaseToCsv</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>
 
  <name>SpringBatchDatabaseToCsv</name>
 
    <properties>
        <springframework.version>4.0.6.RELEASE</springframework.version>
        <springbatch.version>3.0.1.RELEASE</springbatch.version>
        <mysql.connector.version>5.1.31</mysql.connector.version>
        <joda-time.version>2.3</joda-time.version>
    </properties>
 
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${springframework.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>${springframework.version}</version>
        </dependency>
 
        <dependency>
            <groupId>org.springframework.batch</groupId>
            <artifactId>spring-batch-core</artifactId>
            <version>${springbatch.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.batch</groupId>
            <artifactId>spring-batch-infrastructure</artifactId>
            <version>${springbatch.version}</version>
        </dependency>
         
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.connector.version}</version>
        </dependency>
         
        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>${joda-time.version}</version>
        </dependency>
    </dependencies>
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.2</version>
                    <configuration>
                        <source>1.6</source>
                        <target>1.6</target>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
 
</project>
As we need to interact with db this time, we will use spring-jdbc support. We will also need mysql connector to communicate with MySQL, and since we are also using joda-time for any date-time processing we might need, we will include that dependency as well.

Step 4: Create domain object & Mapper (RowMapper implementaion)

We will be mapping the data from database table to properties of our domain object.

com.websystique.springbatch.model.ExamResult

package com.websystique.springbatch.model;
 
import org.joda.time.LocalDate;
 
 
public class ExamResult {
     
    private String studentName;
    private LocalDate dob;
    private double percentage;
     
 
    public String getStudentName() {
        return studentName;
    }
     
    public void setStudentName(String studentName) {
        this.studentName = studentName;
    }
     
    public LocalDate getDob() {
        return dob;
    }
     
    public void setDob(LocalDate dob) {
        this.dob = dob;
    }
     
    public double getPercentage() {
        return percentage;
    }
     
    public void setPercentage(double percentage) {
        this.percentage = percentage;
    }
 
    @Override
    public String toString() {
        return "ExamResult [studentName=" + studentName + ", dob=" + dob + ", percentage=" + percentage + "]";
    }
     
     
}
Below class will eventually map the data from database into domain object based on actual properties datatypes.

com.websystique.springbatch.ExamResultRowMapper

package com.websystique.springbatch;
 
import java.sql.ResultSet;
import java.sql.SQLException;
 
import org.joda.time.LocalDate;
import org.springframework.jdbc.core.RowMapper;
 
import com.websystique.springbatch.model.ExamResult;
 
public class ExamResultRowMapper implements RowMapper<ExamResult>{
 
    @Override
    public ExamResult mapRow(ResultSet rs, int rowNum) throws SQLException {
 
        ExamResult result = new ExamResult();
        result.setStudentName(rs.getString("student_name"));
        result.setDob(new LocalDate(rs.getDate("dob")));
        result.setPercentage(rs.getDouble("percentage"));
             
        return result;
    } 
 
}
Step 5: Create an ItemProcessor

ItemProcessor is Optional, and called after item read but before item write. It gives us the opportunity to perform a business logic on each item. In our case, for example, we will filter out all the items whose percentage is less than 80. So final result will only have records with percentage >= 80.

com.websystique.springbatch.ExamResultItemProcessor

package com.websystique.springbatch;
 
import org.springframework.batch.item.ItemProcessor;
 
import com.websystique.springbatch.model.ExamResult;
 
public class ExamResultItemProcessor implements ItemProcessor<ExamResult, ExamResult>{
 
    @Override
    public ExamResult process(ExamResult result) throws Exception {
        System.out.println("Processing result :"+result);
 
        /*
         * Only return results which are equal or more than 80%
         *
         */
        if(result.getPercentage() < 80){
            return null;
        }
 
        return result;
    }
 
}
Step 6: Add a Job listener(JobExecutionListener)

Job listener is Optional and provide the opportunity to execute some business logic before job start and after job completed.For example setting up environment can be done before job and cleanup can be done after job completed.


 
com.websystique.springbatch.ExamResultJobListener

package com.websystique.springbatch;
 
import java.util.List;
 
import org.joda.time.DateTime;
import org.springframework.batch.core.BatchStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
 
public class ExamResultJobListener implements JobExecutionListener{
 
    private DateTime startTime, stopTime;
 
    @Override
    public void beforeJob(JobExecution jobExecution) {
        startTime = new DateTime();
        System.out.println("ExamResult Job starts at :"+startTime);
    }
 
    @Override
    public void afterJob(JobExecution jobExecution) {
        stopTime = new DateTime();
        System.out.println("ExamResult Job stops at :"+stopTime);
        System.out.println("Total time take in millis :"+getTimeInMillis(startTime , stopTime));
 
        if(jobExecution.getStatus() == BatchStatus.COMPLETED){
            System.out.println("ExamResult job completed successfully");
            //Here you can perform some other business logic like cleanup
        }else if(jobExecution.getStatus() == BatchStatus.FAILED){
            System.out.println("ExamResult job failed with following exceptions ");
            List<Throwable> exceptionList = jobExecution.getAllFailureExceptions();
            for(Throwable th : exceptionList){
                System.err.println("exception :" +th.getLocalizedMessage());
            }
        }
    }
 
    private long getTimeInMillis(DateTime start, DateTime stop){
        return stop.getMillis() - start.getMillis();
    }
 
}
Step 7: Create Spring Context with job configuration

Create dataSource bean needed for database communication

src/main/resources/context-datasource.xml

<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:batch="http://www.springframework.org/schema/batch" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans 
 
http://www.springframework.org/schema/beans/spring-beans-4.0.xsd">
 
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/websystique" />
        <property name="username" value="myuser" />
        <property name="password" value="mypassword" />
    </bean>
 
</beans>          
Create the Spring context with batch job configuration.

src/main/resources/spring-batch-context.xml

<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:batch="http://www.springframework.org/schema/batch" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/batch http://www.springframework.org/schema/batch/spring-batch-3.0.xsd
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd">
 
    <import resource="classpath:context-datasource.xml" />
 
    <!-- JobRepository and JobLauncher are configuration/setup classes -->
    <bean id="jobRepository"
        class="org.springframework.batch.core.repository.support.MapJobRepositoryFactoryBean" />
 
    <bean id="jobLauncher"
        class="org.springframework.batch.core.launch.support.SimpleJobLauncher">
        <property name="jobRepository" ref="jobRepository" />
    </bean>
 
 
    <!-- ItemReader which reads from database and returns the row mapped by 
        rowMapper -->
    <bean id="databaseItemReader"
        class="org.springframework.batch.item.database.JdbcCursorItemReader">
 
        <property name="dataSource" ref="dataSource" />
 
        <property name="sql"
            value="SELECT STUDENT_NAME, DOB, PERCENTAGE FROM EXAM_RESULT" />
 
        <property name="rowMapper">
            <bean class="com.websystique.springbatch.ExamResultRowMapper" />
        </property>
 
    </bean>
 
 
    <!-- ItemWriter writes a line into output flat file -->
    <bean id="flatFileItemWriter" class="org.springframework.batch.item.file.FlatFileItemWriter"
        scope="step">
 
        <property name="resource" value="file:csv/examResult.txt" />
 
        <property name="lineAggregator">
 
            <!-- An Aggregator which converts an object into delimited list of strings -->
            <bean
                class="org.springframework.batch.item.file.transform.DelimitedLineAggregator">
 
                <property name="delimiter" value="|" />
 
                <property name="fieldExtractor">
 
                    <!-- Extractor which returns the value of beans property through reflection -->
                    <bean
                        class="org.springframework.batch.item.file.transform.BeanWrapperFieldExtractor">
                        <property name="names" value="studentName, percentage, dob" />
                    </bean>
                </property>
            </bean>
        </property>
    </bean>
 
 
    <!-- Optional JobExecutionListener to perform business logic before and after the job -->
    <bean id="jobListener" class="com.websystique.springbatch.ExamResultJobListener" />
 
    <!-- Optional ItemProcessor to perform business logic/filtering on the input records -->
    <bean id="itemProcessor" class="com.websystique.springbatch.ExamResultItemProcessor" />
 
    <!-- Step will need a transaction manager -->
    <bean id="transactionManager"
        class="org.springframework.batch.support.transaction.ResourcelessTransactionManager" />
 
    <!-- Actual Job -->
    <batch:job id="examResultJob">
        <batch:step id="step1">
            <batch:tasklet transaction-manager="transactionManager">
                <batch:chunk reader="databaseItemReader" writer="flatFileItemWriter"
                    processor="itemProcessor" commit-interval="10" />
            </batch:tasklet>
        </batch:step>
        <batch:listeners>
            <batch:listener ref="jobListener" />
        </batch:listeners>
    </batch:job>
 
</beans>          
As you can see, we have setup a job with only one step. Step uses JdbcCursorItemReader to read the records from MySQL database, itemProcessor to process the record and FlatFileItemWriter to write the records to a flat file. commit-interval specifies the number of items that can be processed before the transaction is committed/ before the write will happen.Grouping several record in single transaction and write them as chunk provides performance improvement. We have also shown the use of jobListener which can contain any arbitrary logic you might need to run before and after the job.

Step 8: Create Main application to finally run the job

Create a Java application to run the job.

com.websystique.springbatch.Main

package com.websystique.springbatch;
 
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionException;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
 
public class Main {
 
    @SuppressWarnings("resource")
    public static void main(String areg[]){
         
        ApplicationContext context = new ClassPathXmlApplicationContext("spring-batch-context.xml");
         
        JobLauncher jobLauncher = (JobLauncher) context.getBean("jobLauncher");
        Job job = (Job) context.getBean("examResultJob");
      
        try {
            JobExecution execution = jobLauncher.run(job, new JobParameters());
            System.out.println("Job Exit Status : "+ execution.getStatus());
      
        } catch (JobExecutionException e) {
            System.out.println("Job ExamResult failed");
            e.printStackTrace();
        }
    }
 
}
Running above program as java application, you will see following output

INFO: Job: [FlowJob: [name=examResultJob]] launched with the following parameters: [{}]
ExamResult Job starts at :2014-08-04T22:30:14.166+02:00
Aug 4, 2014 10:30:14 PM org.springframework.batch.core.job.SimpleStepHandler handleStep
INFO: Executing step: [step1]
Processing result :ExamResult [studentName=Brian Burlet, dob=1985-02-01, percentage=76.0]
Processing result :ExamResult [studentName=Rita Paul, dob=1993-02-01, percentage=92.0]
Processing result :ExamResult [studentName=Han Yenn, dob=1965-02-01, percentage=83.0]
Processing result :ExamResult [studentName=Peter Pan, dob=1987-02-03, percentage=62.0]
ExamResult Job stops at :2014-08-04T22:30:14.541+02:00
Total time take in millis :375
ExamResult job completed successfully
Aug 4, 2014 10:30:14 PM org.springframework.batch.core.launch.support.SimpleJobLauncher run
INFO: Job: [FlowJob: [name=examResultJob]] completed with the following parameters: [{}] and the following status: [COMPLETED]
Job Exit Status : COMPLETED
You can see that we have processed all input records from Database. Below is the generated flat file (txt) found in project/csv folder

Rita Paul|92.0|1993-02-01
Han Yenn|83.0|1965-02-01
Only the records which are meeting specific condition ( percentage >=80 ) are included here, thanks to itemProcessor filtering logic.

That’s it.

Download Source Code

Download Now!


References

Spring Batch Reference
