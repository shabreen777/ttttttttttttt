db.classname=com.mysql.jdbc.Driver
db.url=jdbc:mysql://localhost:3306/testbase
db.username=ritam
db.password=password
import java.io.FileInputStream;
import java.sql.Connection;
import java.sql.DriverManager;
import java.util.Properties;

public class DBHandler {
	public Connection establishConnection() {
		try {
			FileInputStream fileInputStream = new FileInputStream("db.properties");
			Properties properties = new Properties();
			properties.load(fileInputStream);

			Class.forName(properties.getProperty("db.classname"));

			return DriverManager.getConnection(
					properties.getProperty("db.url"),
					properties.getProperty("db.username"),
					properties.getProperty("db.password")
			);
		} catch (Exception e) {
			e.printStackTrace();
		}

		return null;
	}
}
public class ElectricityBill {
    private String consumerNumber;
    private String consumerName;
    private String consumerAddress;
    private int unitsConsumed;
    
    
    
    
    
    
    
    
    
    
    
    
    private double billAmount;

    public String getConsumerNumber() {
        return consumerNumber;
    }

    public void setConsumerNumber(String consumerNumber) {
        this.consumerNumber = consumerNumber;
    }


    public String getConsumerName() {
        return consumerName;
    }

    public void setConsumerName(String consumerName) {
        this.consumerName = consumerName;
    }

    public String getConsumerAddress() {
        return consumerAddress;
    }

    public void setConsumerAddress(String consumerAddress) {
        this.consumerAddress = consumerAddress;
    }

    public int getUnitsConsumed() {
        return unitsConsumed;
    }

    public void setUnitsConsumed(int unitsConsumed) {
        this.unitsConsumed = unitsConsumed;
    }

    public double getBillAmount() {
        return billAmount;
    }

    public void setBillAmount(double billAmount) {
        this.billAmount = billAmount;
    }

    public void calculateBillAmount() {
        billAmount = 0;
        int tempUnits = unitsConsumed;


        if (tempUnits > 100) {
            tempUnits -= 100;

            if (tempUnits > 200) {
                tempUnits -= 200;
                billAmount += 200 * 1.5;

                if (tempUnits > 300) {
                    tempUnits -= 300;
                    billAmount += 300 * 3.5;

                    if (tempUnits > 400) {
                        tempUnits -= 400;
                        billAmount += 400 * 5.5;
                        billAmount += tempUnits * 7.5;
                    } else {
                        billAmount += tempUnits * 5.5;
                    }
                } else {
                    billAmount += tempUnits * 3.5;
                }
            } else {
                billAmount += tempUnits * 1.5;
            }
        }
    }
}
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class ElectricityBoard {
    public boolean validate(String consumerNumber) throws InvalidConsumerNumberException {
        Pattern pattern = Pattern.compile("^0\\d{9}$");
        Matcher matcher = pattern.matcher(consumerNumber);

        if (matcher.matches()) {
            return true;
        } else {
            throw new InvalidConsumerNumberException("Invalid Consumer Number");
        }
    }


    public void addBill(List<ElectricityBill> billList) {
        Connection connection = new DBHandler().establishConnection();

        try {
            for (ElectricityBill bill : billList) {
                PreparedStatement preparedStatement = connection.prepareStatement("insert into ElectricityBill values(?,?,?,?,?);");
                preparedStatement.setString(1, bill.getConsumerNumber());
                preparedStatement.setString(2, bill.getConsumerName());
                preparedStatement.setString(3, bill.getConsumerAddress());
                preparedStatement.setInt(4, bill.getUnitsConsumed());
                preparedStatement.setFloat(5, (float) bill.getBillAmount());

                int result = preparedStatement.executeUpdate();
            }
        } catch (SQLException e) {
            System.out.println(e.getMessage());
        }
    }

    public List<ElectricityBill> generateBill(String filePath) {
        List<ElectricityBill> electricityBills = new ArrayList<>();

        try {
            Scanner scanner = new Scanner(new BufferedReader(new FileReader(filePath)));

            while (scanner.hasNext()) {
                String[] inputs = scanner.nextLine().split(",");

                try {
                    String consumerNumber = inputs[0];
                    boolean validConsumerNumber = validate(consumerNumber);

                    if (validConsumerNumber) {
                        String consumerName = inputs[1];
                        String consumerAddress = inputs[2];
                        int unitsConsumed = Integer.parseInt(inputs[3]);

                        ElectricityBill electricityBill = new ElectricityBill();
                        electricityBill.setConsumerNumber(consumerNumber);
                        electricityBill.setConsumerName(consumerName);
                        electricityBill.setConsumerAddress(consumerAddress);
                        electricityBill.setUnitsConsumed(unitsConsumed);
                        electricityBill.calculateBillAmount();
                        electricityBills.add(electricityBill);
                    }
                } catch (InvalidConsumerNumberException e) {
                    System.out.println(e.getMessage());
                }
            }
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }

        return electricityBills;
    }

}
public class InvalidConsumerNumberException extends Exception {
    public InvalidConsumerNumberException(String message) {
        super(message);
    }
}
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.List;

public class Main {
    public static void main(String[] args) throws SQLException {
        ElectricityBoard electricityBoard = new ElectricityBoard();

        List<ElectricityBill> billList = electricityBoard.generateBill("ElectricityBill.txt");

        System.out.println("Bills parsed from file...");
        for (ElectricityBill bill : billList) {
            System.out.println(String.format("id: %s, name: %s, address: %s, units: %d, bill: %f",
                    bill.getConsumerNumber(),
                    bill.getConsumerName(),
                    bill.getConsumerAddress(),
                    bill.getUnitsConsumed(),
                    bill.getBillAmount())
            );
        }

        // Adding bills to the database
        electricityBoard.addBill(billList);

        Connection connection = new DBHandler().establishConnection();
        Statement statement = connection.createStatement();
        ResultSet resultSet = statement.executeQuery("select * from ElectricityBill");

        System.out.println("Bills retrieved from database ElectricityBill...");

        while (resultSet.next()) {
            String consumerNumber = resultSet.getString(1);
            String consumerName = resultSet.getString(2);
            String consumerAddress = resultSet.getString(3);
            int unitsConsumed = resultSet.getInt(4);
            float billAmount = resultSet.getFloat(5);

            System.out.println(String.format("id: %s, name: %s, address: %s, units: %d, bill: %f",
                    consumerNumber,
                    consumerName,
                    consumerAddress,
                    unitsConsumed,
                    billAmount)
            );
        }
    }
    0191919191,John,Chennai,650
0191919192,Peter,Mumbai,1100
1919191919,Rose,Mumbai,453
0191919193,Tom,Hyderabad,750
01919191945,Raj,Chennai,120
0191919194,Sam,Chennai,250
0191919195,Anya,Chennai,34
Beans.xml--------------------------
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:p="http://www.springframework.org/schema/p"
xmlns:aop="http://www.springframework.org/schema/aop"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:jee="http://www.springframework.org/schema/jee"
xmlns:tx="http://www.springframework.org/schema/tx"
xmlns:task="http://www.springframework.org/schema/task"
xsi:schemaLocation="http://www.springframework.org/schema/aop
http://www.springframework.org/schema/aop/spring-aop-3.2.xsd
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-3.2.xsd
http://www.springframework.org/schema/jee
http://www.springframework.org/schema/jee/spring-jee-3.2.xsd
http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/springtx-3.2.xsd http://www.springframework.org/schema/task
http://www.springframework.org/schema/task/spring-task-3.2.xsd">
<context:property-placeholder location="classpath:charges.properties" />
<bean name="courier" class="com.spring.model.Courier">
<property name="chargePerKg" value="${chargePerKg}" />
<property name="serviceCharge">
<bean class="com.spring.model.ServiceChargeInfo">
<property name="locationServiceCharge">
<map>
<entry key="Coimbatore" value="200.0" />
<entry key="Chennai" value="300.0" />
<entry key="Madurai" value="150.0" />
</map>
</property>
</bean>
</property>
</bean>
<bean id="courierBoObj" class="com.spring.bo.CourierBO" />
<bean name="courierService" class="com.spring.service.CourierService">
<property name="cBoObj" ref="courierBoObj"/>
</bean>
</beans>
Class courierService---------------
package com.spring.service;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.FileSystemXmlApplicationContext;
import com.spring.bo.CourierBO;
import com.spring.exception.InvalidParcelWeightException;
import com.spring.model.Courier;
public class CourierService {
private CourierBO cBoObj;
public CourierBO getcBoObj() {
return cBoObj;
}
public void setcBoObj(CourierBO cBoObj) {
this.cBoObj = cBoObj;
}
public double calculateCourierCharge(int courierId,int weight,String city) throws
InvalidParcelWeightException {
double courierCharge=0.0;
//fill your code
if(weight > 0 && weight < 1000){
ApplicationContext context = new
FileSystemXmlApplicationContext("src/main/resources/beans.xml");
Courier courier = context.getBean("courier", Courier.class);
courier.setCourierId(courierId);
courier.setWeight(weight);
courierCharge = cBoObj.calculateCourierCharge(courier, city);
}else{
throw new InvalidParcelWeightException("Invalid Parcel Weight");
}
return courierCharge;
}
}
Class serviceCharge--------------------------
package com.spring.model;
import java.util.Map;
public class ServiceChargeInfo {
private Map<String,Float> locationServiceCharge;
public Map<String, Float> getLocationServiceCharge() {
return locationServiceCharge;
}
public void setLocationServiceCharge(Map<String, Float> locationServiceCharge) {
this.locationServiceCharge = locationServiceCharge;
}
}
Class courier------------------
package com.spring.model;
public class Courier {
private int courierId;
private int weight;
private float chargePerKg;
private ServiceChargeInfo serviceCharge;
public ServiceChargeInfo getServiceCharge() {
return serviceCharge;
}
public void setServiceCharge(ServiceChargeInfo serviceCharge) {
this.serviceCharge = serviceCharge;
}
public int getCourierId() {
return courierId;
}
public void setCourierId(int courierId) {
this.courierId = courierId;
}
public int getWeight() {
return weight;
}
public void setWeight(int weight) {
this.weight = weight;
}
public float getChargePerKg() {
return chargePerKg;
}
public void setChargePerKg(float chargePerKg) {
this.chargePerKg = chargePerKg;
}
}
Class driver----------------------------------
package com.spring.main;
import java.util.Scanner;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.FileSystemXmlApplicationContext;
import com.spring.exception.InvalidParcelWeightException;
import com.spring.service.CourierService;
public class Driver {
public static void main(String[] args) throws InvalidParcelWeightException {

//fill the code
Scanner sc = new Scanner(System.in);
ApplicationContext context = new
FileSystemXmlApplicationContext("src/main/resources/beans.xml");
CourierService service = context.getBean("courierService", CourierService.class );
int courierId = 0;
int weight = 0;
String cityName = "";
double courierCharge=0.0;
System.out.println("Enter the courier ID:");
courierId = sc.nextInt();
System.out.println("Enter the total weight of parcel:");
weight = sc.nextInt();
sc.nextLine();
System.out.println("Enter the city:");
cityName = sc.nextLine();
courierCharge = service.calculateCourierCharge(courierId, weight, cityName);
System.out.println("Total Courier Charge: "+courierCharge);
}
}
Exception class--------------------------
package com.spring.exception;
public class InvalidParcelWeightException extends Exception {
public InvalidParcelWeightException(String msg) {
super(msg);
}
}
Class courierBo--------------
package com.spring.bo;
import java.util.Map;
import com.spring.model.Courier;
public class CourierBO {
public double calculateCourierCharge(Courier cObj, String city) {
double courierCharge = 0.0;
Map<String, Float> map = cObj.getServiceCharge().getLocationServiceCharge();
// fill the code
courierCharge = cObj.getWeight() * cObj.getChargePerKg();
if (map.containsKey(city)) {
courierCharge = courierCharge +
cObj.getServiceCharge().getLocationServiceCharge().get(city);
}
return courierCharge;
}
}
https://onlinegdb.com/r1-4TEgfu
https://www.englishforums.com/English/LetAnswersCorrect/qlnnk/post.htm
What?                                                                                                                      
A scam is a used to cheat someone out of something, especially 
 scams can affect the economy 
of the nation as a whole and also damage the world economy.
                                                                             SCAM IN 
                                                                       INDIAN ECONOMY
When?
India has been home to several scams post 1992, resulting to losses 
amounting to lakhs of crores of rupees to the economy
1)In 2016,  Vijay Mallya – Rs. 9000 Crore
2)In 2004, Coalgate Scam – Rs. 1.86 lakh crore
3)In 2012, 2G Spectrum scam – Rs. 1,76,000 crore
4)In 2010, Commonwealth Games scam – Rs. 70,000 crore
![Uploading image.png…]()
book a movie ticket
bookamovie.java
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class BookAMovie {
	private List<MovieTicket> movieTicketList = new ArrayList<>();

	public List<MovieTicket> getMovieTicketList() {
		return movieTicketList;
	}

	public void setMovieTicketList(List<MovieTicket> movieTicketList) {
		this.movieTicketList = movieTicketList;
	}

	public boolean validateCircle(String circle) throws InvalidMovieTicketException {
		if (circle.equalsIgnoreCase("king") || circle.equalsIgnoreCase("queen"))
			return true;
		else
			throw new InvalidMovieTicketException("Invalid circle");
	}

	public boolean addMovieTicket(int ticketId, String movieName, int screenNumber, int numberOfSeats, String circle,
			LocalDate showDate) throws InvalidMovieTicketException {
		if (validateCircle(circle)) {
			movieTicketList.add(new MovieTicket(ticketId, movieName, screenNumber, numberOfSeats, circle, showDate));
			return true;
		}

		throw new InvalidMovieTicketException("Invalid circle");
	}

	public MovieTicket viewMovieTicketById(int ticketId) throws InvalidMovieTicketException {
		for (MovieTicket ticketObj : movieTicketList) {
			if (ticketObj.getTicketId() == ticketId) {
				return ticketObj;
			}
		}

		throw new InvalidMovieTicketException("Invalid movie ticket id");
	}

	public List<MovieTicket> viewMovieTicketByScreen(int screenNumber) {
		List<MovieTicket> temp = new ArrayList<MovieTicket>();

		for (MovieTicket ticketObj : movieTicketList) {
			if (ticketObj.getScreenNumber() == screenNumber)
				temp.add(ticketObj);
		}
		return temp;
	}

	public Map<String, List<MovieTicket>> viewTicketsMovieWise() {
		Map<String, List<MovieTicket>> temp = new LinkedHashMap<String, List<MovieTicket>>();

		for (MovieTicket ticketObj : movieTicketList) {
			String movieName = ticketObj.getMovieName();
			
			if (!temp.containsKey(movieName)) {
				temp.put(movieName, new ArrayList<MovieTicket>());
			}
				
			temp.get(movieName).add(ticketObj);
		}
		
		return temp;
	}

	public Map<Integer, Integer> viewScreenWiseTotalSeatsBooked(LocalDate fromDate, LocalDate toDate) {
		Map<Integer, Integer> temp = new LinkedHashMap<Integer, Integer>();

		for (MovieTicket ticketObj : movieTicketList) {
			if (ticketObj.getShowDate().compareTo(fromDate) >= 0 && ticketObj.getShowDate().compareTo(toDate) <= 0) {
				int screenNumber = ticketObj.getScreenNumber();

				if (!temp.containsKey(screenNumber)) {
					temp.put(screenNumber, 0);
				}

				temp.put(screenNumber, temp.get(screenNumber) + ticketObj.getNumberOfSeats());
			}

		}

		return temp;
	}
}


public class InvalidMovieTicketException extends Exception {
	public InvalidMovieTicketException(String message) {
		super(message);
	}
}
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class MovieTicket {
	private int ticketId;
	private String movieName;
	private int screenNumber;
	private int numberOfSeats;
	private String circle;
	private LocalDate showDate;

	public MovieTicket() {

	}

	public MovieTicket(int ticketId, String movieName, int screenNumber, int numberOfSeats, String circle,
			LocalDate showDate) {
		this.ticketId = ticketId;
		this.movieName = movieName;
		this.screenNumber = screenNumber;
		this.numberOfSeats = numberOfSeats;
		this.circle = circle;
		this.showDate = showDate;
	}

	public int getTicketId() {
		return ticketId;
	}

	public void setTicketId(int ticketId) {
		this.ticketId = ticketId;
	}

	public String getMovieName() {
		return movieName;
	}

	public void setMovieName(String movieName) {
		this.movieName = movieName;
	}

	public int getScreenNumber() {
		return screenNumber;
	}

	public void setScreenNumber(int screenNumber) {
		this.screenNumber = screenNumber;
	}

	public int getNumberOfSeats() {
		return numberOfSeats;
	}

	public void setNumberOfSeats(int numberOfSeats) {
		this.numberOfSeats = numberOfSeats;
	}

	public String getCircle() {
		return circle;
	}

	public void setCircle(String circle) {
		this.circle = circle;
	}

	public LocalDate getShowDate() {
		return showDate;
	}

	public void setShowDate(LocalDate showDate) {
		this.showDate = showDate;
	}

	@Override
	public String toString() {
		String date = showDate.format(DateTimeFormatter.ofPattern("dd-MM-yyyy"));

		return "MovieTicket [ticketId=" + ticketId + ", movieName=" + movieName + ", screenNumber=" + screenNumber
				+ ", numberOfSeats=" + numberOfSeats + ", circle=" + circle + ", showDate=" + date + "]";
	}
}
import static org.junit.Assert.*;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import org.junit.Rule;
import org.junit.rules.ExpectedException;
import org.junit.Before;
import org.junit.FixMethodOrder;
import org.junit.Test;
import org.junit.runners.MethodSorters;
import java.util.Map;

@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class BookAMovieTest {
	@Rule
	public ExpectedException exceptionRule = ExpectedException.none();

	static BookAMovie bookAMovie;
	static MovieTicket t1;
	static MovieTicket t2;
	static MovieTicket t3;
	static MovieTicket t4;
	static MovieTicket t5;
	static MovieTicket t6;

	@Before
	public void setUp() throws Exception {
		// Try to create few MovieTicket objects and add to a list.
		// Set that list to movieTicketList in BookAMovie using setMovieTicketList
		// method
		bookAMovie = new BookAMovie();
		
		t1 = new MovieTicket(123, "Avenger", 2, 5, "king", LocalDate.parse("2017-11-10"));
		t2 = new MovieTicket(124, "Iron Man", 1, 5, "queen", LocalDate.parse("2020-08-13"));
		t3 = new MovieTicket(125, "Captain Marvel", 3, 4, "king", LocalDate.parse("2020-08-23"));
		t4 = new MovieTicket(126, "Avenger EndGame", 1, 7, "queen", LocalDate.parse("2020-09-24"));
		
		List<MovieTicket> movieTicketList2 = new ArrayList<>();
		movieTicketList2.add(t1);
		movieTicketList2.add(t2);
		movieTicketList2.add(t3);
		movieTicketList2.add(t4);

		bookAMovie.setMovieTicketList(movieTicketList2);
	}

	@Test
	public void test11ValidateCircleWhenKing() throws InvalidMovieTicketException {
		// test the validateCircle method when a valid circle �king� is provided
		assertTrue(bookAMovie.validateCircle("king"));
	}

	@Test
	public void test12ValidateCircleWhenQueen() throws InvalidMovieTicketException {
		// test the validateCircle method when a valid circle �queen� is provided.
		assertTrue(bookAMovie.validateCircle("queen"));
	}

	@Test
	public void test13ValidateCircleWhenInvalid() throws InvalidMovieTicketException {
		// test the validateCircle method when an invalid circle is passed to this
		// method
		
		exceptionRule.expect(InvalidMovieTicketException.class);
		exceptionRule.expectMessage("Invalid circle");
		bookAMovie.validateCircle("joker");
	}

	@Test
	public void test14AddMovieTicketForValidCircle() throws InvalidMovieTicketException {
		// test the addMovieTicket method when valid circle is provided for the
		// MovieTicket
		assertTrue(bookAMovie.addMovieTicket(123, "Avenger", 2, 5, "king", LocalDate.parse("2017-11-10")));
	}

	@Test
	public void test15AddMovieTicketForInvalidCircle() throws InvalidMovieTicketException {
		// test the addMovieTicket method when invalid circle is provided for the
		// MovieTicket
		exceptionRule.expect(InvalidMovieTicketException.class);
		exceptionRule.expectMessage("Invalid circle");
		
		bookAMovie.addMovieTicket(129, "Captain Marvel", 3, 4, "kingqueen", LocalDate.parse("2020-08-19"));
	}

	@Test
	public void test16ViewMovieTicketByIdForValidId() throws InvalidMovieTicketException {
		// test the viewMovieTicketById method when a ticketId is passed as parameter
		// exists in
		// the movieTicketList
		assertEquals(t1, bookAMovie.viewMovieTicketById(123));
	}

	@Test
	public void test17ViewMovieTicketByIdForInvalidId() throws InvalidMovieTicketException {
		// test the viewMovieTicketById method when a ticketId is passed as parameter
		// does not exist in the movieTicketList
		exceptionRule.expect(InvalidMovieTicketException.class);
		exceptionRule.expectMessage("Invalid movie ticket id");
		
		bookAMovie.viewMovieTicketById(12345);
	}

	@Test
	public void test18ViewMovieTicketByScreen() {
		// test the viewMovieTicketByScreen method
		List<MovieTicket> tmp = new ArrayList<MovieTicket>();
		tmp.add(t2);
		tmp.add(t4);
		
		assertEquals(tmp, bookAMovie.viewMovieTicketByScreen(1));
	}

	@Test
	public void test19VewTicketsMovieWise() {
		// test the viewTicketsMovieWise method
		Map<String, List<MovieTicket>> map = bookAMovie.viewTicketsMovieWise();
		int l = map.size();
		
		assertEquals(4, l);
	}

	@Test
	public void test20ViewScreenWiseTotalSeatsBooked() {
		// test the viewScreenWiseTotalSeatsBooked method
		Map<Integer, Integer> map = bookAMovie.viewScreenWiseTotalSeatsBooked(LocalDate.parse("2017-10-25"),
				LocalDate.now());
		
		assertFalse(map.isEmpty());
	}
}
import org.junit.runner.JUnitCore;
import org.junit.runner.Result;
import org.junit.runner.notification.Failure;

public class Main {
	public static void main(String args[]) {
		Result result = JUnitCore.runClasses(BookAMovieTest.class);

		if (result.getFailureCount() == 0) {
			System.out.println("No Failures");
		} else {
			for (Failure failure : result.getFailures()) {
				System.out.println(failure.toString());
			}
		}
		
		System.out.println("Result " + result.wasSuccessful());
	}
}
pom.xml 
<?xml version="1.0" encoding="UTF-8"?>

-<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0">

<modelVersion>4.0.0</modelVersion>

<groupId>dev.ritam</groupId>

<artifactId>movie_ticket_booking</artifactId>

<version>0.0.1-SNAPSHOT</version>

<name>movie_ticket_booking</name>

<!-- FIXME change it to the project's website -->


<url>http://www.example.com</url>


-<properties>

<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

<maven.compiler.source>1.7</maven.compiler.source>

<maven.compiler.target>1.7</maven.compiler.target>

</properties>


-<dependencies>


-<dependency>

<groupId>junit</groupId>

<artifactId>junit</artifactId>

<version>4.11</version>

<scope>test</scope>

</dependency>

</dependencies>


-<build>


-<pluginManagement>

<!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->



-<plugins>

<!-- clean lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#clean_Lifecycle -->



-<plugin>

<artifactId>maven-clean-plugin</artifactId>

<version>3.1.0</version>

</plugin>

<!-- default lifecycle, jar packaging: see https://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_jar_packaging -->



-<plugin>

<artifactId>maven-resources-plugin</artifactId>

<version>3.0.2</version>

</plugin>


-<plugin>

<artifactId>maven-compiler-plugin</artifactId>

<version>3.8.0</version>

</plugin>


-<plugin>

<artifactId>maven-surefire-plugin</artifactId>

<version>2.22.1</version>

</plugin>


-<plugin>

<artifactId>maven-jar-plugin</artifactId>

<version>3.0.2</version>

</plugin>


-<plugin>

<artifactId>maven-install-plugin</artifactId>

<version>2.5.2</version>

</plugin>


-<plugin>

<artifactId>maven-deploy-plugin</artifactId>

<version>2.8.2</version>

</plugin>

<!-- site lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#site_Lifecycle -->



-<plugin>

<artifactId>maven-site-plugin</artifactId>

<version>3.7.1</version>

</plugin>


-<plugin>

<artifactId>maven-project-info-reports-plugin</artifactId>

<version>3.0.0</version>

</plugin>

</plugins>

</pluginManagement>

</build>

</project>
project refraction code
Employee.java

package project;

public class Employee {
	 private String employeeId;
	 private String employeeName;
	 private String emailId;
	 private String designation;
	 
	public String getEmployeeId() {
		return employeeId;
	}
	public String getEmployeeName() {
		return employeeName;
	}
	public String getEmailId() {
		return emailId;
	}
	public String getDesignation() {
	s	return designation;
	}
	public Employee(String employeeId, String employeeName, String emailId, String designation) {
		super();
		this.employeeId = employeeId;
		this.employeeName = employeeName;
		this.emailId = emailId;
		this.designation = designation;
	}
}

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
EmployeeDAO.java

package project;

import java.util.ArrayList;

public class EmployeeDAO {
	
	ArrayList<Employee> employeeList =  new ArrayList<Employee>();
	
	public void addEmployee(Employee obj){
		employeeList.add(obj);
	}
	
	public void removeEmployee(Employee obj)
	{
		employeeList.remove(obj);
	}
	public void viewEmployee()
	{
		for(int i=0;i<employeeList.size();i++)
		{
			System.out.println("Employee Id:" + employeeList.get(i).getEmployeeId());
			System.out.println("Employee Name:" + employeeList.get(i).getEmployeeName());
			System.out.println("Email Id:" + employeeList.get(i).getEmailId());
			System.out.println("Designation: "+ employeeList.get(i).getDesignation());
			
		}
	}
}
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Project.java

package project;

public class Project{
	private String projectId;
	private String projectName;
	private String projectManagerName;
	private int duration;
	private String startDate;
	private String endDate;
	
	public Project(){
	// auto constructor stub
	}
	public Project(String projectName, String projectManagerName, int duration, String startDate,String endDate) {
		super();
		this.projectName = projectName;
		this.projectManagerName = projectManagerName;
		this.duration = duration;
		this.startDate = startDate;
		this.endDate = endDate;
	}
	public String getProjectId() {
		return projectId;
	}
	public String getProjectName() {
		return projectName;
	}
	public String getProjectManagerName() {
		return projectManagerName;
	}
	public int getDuration() {
		return duration;
	}
	public String getStartDate() {
		return startDate;
	}
	public String getEndDate() {
		return endDate;
	}
	
	@Override
	public String toString(){
		
		return projectId+" "+projectName;
	}
}
	
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ProjectAllocation.java

package project;

import java.util.Date;

public class ProjectAllocation {
	private Employee employee;
	private Project project;
	private int projectAllocationId;
	private String moduleName;
	private static final int NO_OF_PROJECTS_WORKING_IN_PARALLEL = 0;
	private Date allocationDate;
	public static final int NO_OF_HOURS_ALLOCATED = 160;

	public ProjectAllocation() {
		// CONSTRUCTOR STUB
	}

	public ProjectAllocation(Employee employee, Project project, int projectAllocationId, String moduleName,
			Date allocationDate) {
		super();
		this.employee = employee;
		this.project = project;
		this.projectAllocationId = projectAllocationId;
		this.moduleName = moduleName;
		this.allocationDate = allocationDate;
	}

	public Employee getEmployee() {
		return employee;
	}

	public Project getProject() {
		return project;
	}

	public int getProjectAllocationId() {
		return projectAllocationId;
	}

	public String getModuleName() {
		return moduleName;
	}

	public static int getNO_OF_PROJECTS_WORKING_IN_PARALLEL() {
		return NO_OF_PROJECTS_WORKING_IN_PARALLEL;
	}

	public Date getAllocationDate() {
		return allocationDate;
	}

}
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ProjectAllocationDAO.java

package project;

import java.util.ArrayList;

public class ProjectAllocationDAO {
	ArrayList<ProjectAllocation> projectAllocationList = new ArrayList<ProjectAllocation>();

	public void addProjectAllocation(ProjectAllocation obj) {
		projectAllocationList.add(obj);

	}

	public void removeProjectAllocation(ProjectAllocation obj) {
		projectAllocationList.remove(obj);

	}

	public void viewProjectAllocation() {
		if (projectAllocationList.isEmpty()) {
			System.out.println("Project Allocation List is empty");
		} else {
			for (int i = 0; i < projectAllocationList.size(); i++) {
				System.out.println("Project Allocation Id:" + projectAllocationList.get(i).getProjectAllocationId());
				System.out.println("Project Id:" + projectAllocationList.get(i).getProject().getProjectId());
				System.out.println("Employee Id:" + projectAllocationList.get(i).getEmployee().getEmployeeId());
				System.out.println("Allocation Date:" + projectAllocationList.get(i).getAllocationDate());
				System.out.println("Module Name:" + projectAllocationList.get(i).getModuleName());
			}
		}
	}

}
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ProjectDAO.java

package project;

import java.util.ArrayList;

public class ProjectDAO {

	ArrayList<Project> projectList = new ArrayList<Project>();

	public void addProject(Project obj) {
		projectList.add(obj);
	}

	public void removeProject(Project obj) {
		projectList.remove(obj);
	}

	public void viewMember() {
		for (int i = 0; i < projectList.size(); i++) {
			System.out.println("Project Id:" + projectList.get(i).getProjectId());
			System.out.println("Project Name:" + projectList.get(i).getProjectName());
			System.out.println("Project Manager Name:" + projectList.get(i).getProjectManagerName());
			System.out.println("Duration:" + projectList.get(i).getDuration());
			System.out.println("Start Date:" + projectList.get(i).getStartDate());
			System.out.println("End Date:" + projectList.get(i).getEndDate());

		}
	}

}
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
<?xml version="1.0" encoding="UTF-8"?>

<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0">

<modelVersion>4.0.0</modelVersion>

<groupId>dev.ritam</groupId>

<artifactId>movie_ticket_booking</artifactId>

<version>0.0.1-SNAPSHOT</version>

<name>movie_ticket_booking</name>

<!-- FIXME change it to the project's website -->


<url>http://www.example.com</url>


<properties>

<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

<maven.compiler.source>1.7</maven.compiler.source>

<maven.compiler.target>1.7</maven.compiler.target>

</properties>


<dependencies>


<dependency>

<groupId>junit</groupId>

<artifactId>junit</artifactId>

<version>4.11</version>

<scope>test</scope>

</dependency>

</dependencies>


<build>


<pluginManagement>

<!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->



<plugins>

<!-- clean lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#clean_Lifecycle -->



<plugin>

<artifactId>maven-clean-plugin</artifactId>

<version>3.1.0</version>

</plugin>

<!-- default lifecycle, jar packaging: see https://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_jar_packaging -->



<plugin>

<artifactId>maven-resources-plugin</artifactId>

<version>3.0.2</version>

</plugin>


<plugin>

<artifactId>maven-compiler-plugin</artifactId>

<version>3.8.0</version>

</plugin>


<plugin>

<artifactId>maven-surefire-plugin</artifactId>

<version>2.22.1</version>

</plugin>


<plugin>

<artifactId>maven-jar-plugin</artifactId>

<version>3.0.2</version>

</plugin>


<plugin>

<artifactId>maven-install-plugin</artifactId>

<version>2.5.2</version>

</plugin>


<plugin>

<artifactId>maven-deploy-plugin</artifactId>

<version>2.8.2</version>

</plugin>

<!-- site lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#site_Lifecycle -->



<plugin>

<artifactId>maven-site-plugin</artifactId>

<version>3.7.1</version>

</plugin>


<plugin>

</plugins>

</pluginManagement>

</build>

</project>






















TMS Application.java:
package com;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
@SpringBootApplication
@ComponentScan("com.*")
public class TmsApplication {
/**
* Starting point of the application
* 
* @param args Arguments passed to the application
*/
public static void main(String[] args) {
SpringApplication.run(TmsApplication.class, args);
}
}

InternationalizationConfig.java:
package com.controller;import java.util.Locale;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;
@Configuration
public class InternationalizationConfig extends WebMvcConfigurerAdapter {
/**
* Set default Locale
* 
* @return A bean of LocalResolver
*/
@Bean
public LocaleResolver localeResolver() {
SessionLocaleResolver slr = new SessionLocaleResolver();
slr.setDefaultLocale(Locale.US);return slr;
}
/**
* Set path variable name for changing language
* 
* @return A bean of LocaleChangeInterceptor
*/
@Bean
public LocaleChangeInterceptor localeChangeInterceptor() {
LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
lci.setParamName("language");
return lci;
}
/**
* Add interceptor into the registry
*/
@Override
public void addInterceptors(InterceptorRegistry registry) {
registry.addInterceptor(localeChangeInterceptor());
}
/* Set base name for messages.properties files Set default encoding to UTF-8
* 
* @return A bean of MessageSource
*/
@Bean
public MessageSource messageSource() {
ReloadableResourceBundleMessageSource rrbms = new 
ReloadableResourceBundleMessageSource();
rrbms.setBasename("classpath:messages");
rrbms.setDefaultEncoding("UTF-8");
return rrbms;
}
/**
* Set validation message source
* 
* @return A bean of LocalValidatorFactoryBean
*/
@Bean
public LocalValidatorFactoryBean localValidatorFactoryBean() {
LocalValidatorFactoryBean lvfb = new LocalValidatorFactoryBean();
lvfb.setValidationMessageSource(messageSource());
return lvfb;
}}

TaxController.java:
package com.controller;
import java.util.Arrays;
import java.util.List;
import javax.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import com.model.UserClaim;
import com.service.TaxService;
@Controller
public class TaxController {
@Autowired
public TaxService taxService;/**
* Display taxclaim.jsp page when a get request is pushed on url
* /getTaxClaimFormPage
* 
* @param userClaim Is the UserClaim component
* @return taxclaim as a jsp page
* @see UserClaim
*/
@RequestMapping(value = "/getTaxClaimFormPage", method = RequestMethod.GET)
public String claimPage(@ModelAttribute("userClaim") UserClaim userClaim) {
return "taxclaim";
}
/**
* Return result.jsp age when validation is successful Otherwise return back to
* taxclaim page with error message
* 
* @param userClaim UserClaim component
* @param result BindingResult which validate the user input
* @param map ModelMap to put attribute which will be forwarded to next
* page
* @return "result.jsp" page if the validation is successful otherwise
* "taxclaim.jsp" with error included
*/@RequestMapping(value = "/calculateTax", meth
zar
od = RequestMethod.GET)
public String calculateTax(@Valid @ModelAttribute("userClaim") UserClaim userClaim, 
BindingResult result,
ModelMap map) {
if (result.hasErrors()) {
return "taxclaim";
}
double amount = taxService.calculateTax(userClaim);
map.addAttribute("amount", amount);
return "result";
}
/**
* Populate <form:select /> tag in the taxclaim.jsp page
* 
* @return List of expenses
*/
@ModelAttribute("expenseList")
public List<String> populateExpense() {
return Arrays.asList("MedicalExpense", "TravelExpense", "FoodExpense");
}
}

UserClaim.java:
package com.model;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.PositiveOrZero;
import javax.validation.constraints.Size;
import org.springframework.stereotype.Component;
@Component
public class UserClaim {
private String expenseType;
@PositiveOrZero(message = "{error.expenseAmount.negative}")
private double expenseAmt;
@NotBlank(message = "{error.employeeId}")
@Size(min = 5, message = "{error.employeeId.size}")
private String employeeId;
public String getExpenseType() {
return expenseType;
}
public void setExpenseType(String expenseType) {
this.expenseType = expenseType;}
public double getExpenseAmt() {
return expenseAmt;
}
public void setExpenseAmt(double expenseAmt) {
this.expenseAmt = expenseAmt;
}
public String getEmployeeId() {
return employeeId;
}
public void setEmployeeId(String employeeId) {
this.employeeId = employeeId;
}
}

TaxService:
package com.service;
import org.springframework.stereotype.Service;
import com.model.UserClaim;
@Service
public interface TaxService {
/**
* Calculate Tax
* 
* @param userClaim UserClaim bean
* @return Calculated tax
*/
public double calculateTax(UserClaim userClaim);
}

TaxServiceImpl:
package com.service;
import org.springframework.stereotype.Service;
import com.model.UserClaim;
@Service
public class TaxServiceImpl implements TaxService {
/**
* Calculate the tax according to the srs
* 
* @param userClaim UserClaim component to get the values
* @return Calculated tax
*/
@Override
public double calculateTax(UserClaim userClaim) {
String e = userClaim.getExpenseType();
double a = userClaim.getExpenseAmt();
double t = 0.0;
if (e.startsWith("M")) {
if (a <= 1000) {
t = 15.0;} else if (a > 1000 && a <= 10000) {
t = 20.0;
} else if (a > 10000) {
t = 25.0;
}
} else if (e.startsWith("T")) {
if (a <= 1000) {
t = 10.0;
} else if (a > 1000 && a <= 10000) {
t = 15.0;
} else if (a > 10000) {
t = 20.0;
}
} else if (e.startsWith("F")) {
if (a <= 1000) {
t = 5.0;
} else if (a > 1000 && a <= 10000) {
t = 10.0;
} else if (a > 10000) {
t = 15.0;
}
}
return a * (t / 100.0);
} }

Application.properties:
server.port=9095
spring.mvc.view.prefix=/WEB-INF/jsp/
spring.mvc.view.suffix=.jsp
spring.mvc.static-class-path=/resources/**

messade_deproperties:
label.employeeId=Employee ID in German
label.expenseType=Expense Type in German
label.expenseAmount=Expense Amount in German
error.employeeId=Employee ID cannot be empty in German
error.employeeId.size=Employee ID should be at least 5 characters in German
error.expenseAmount=Expense Amount cannot be empty in German
error.expenseAmount.numeric=Expense amount should be numeric only in German
error.expenseAmount.negative=Expense amount should not be a negative number in German

message_fr.properties:
label.employeeId=Employee ID in French
label.expenseType=Expense Type in French
label.expenseAmount=Expense Amount in French
error.employeeId=Employee ID cannot be empty in French
error.employeeId.size=Employee ID should be at least 5 characters in French
error.expenseAmount=Expense Amount cannot be empty in French
error.expenseAmount.numeric=Expense amount should be numeric only in French
error.expenseAmount.negative=Expense amount should not be a negative number in French

message.properties:
label.employeeId=Employee ID in English
label.expenseType=Expense Type in English
label.expenseAmount=Expense Amount in English
error.employeeId=Employee ID cannot be empty in English
error.employeeId.size=Employee ID should be at least 5 characters in English
error.expenseAmount=Expense
zar
Amount cannot be empty in English
error.expenseAmount.numeric=Expense amount should be numeric only in English
error.expenseAmount.negative=Expense amount should not be a negative number in English

result.jsp:
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
pageEncoding="ISO-8859-1"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
"http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
<title>Insert title here</title>
</head>
<body>
<h2>The tax claim for ${ userClaim.expenseType } with expense amount
${ userClaim.expenseAmt } is ${ amount }</h2>
</body>
</html>

Taxclaim.jsp:
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
pageEncoding="ISO-8859-1" isELIgnored="false"%>
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form"%>
<%@ taglib uri="http://www.springframework.org/tags/form" prefix="form"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
"http://www.w3.org/TR/html4/loose.dtd">
<html>
<body style="background-color: lavender">
<h1>
<center>Tax: Tax Claim</center>
</h1>
<a href="/getTaxClaimFormPage?language=en">English</a>|
<a href="/getTaxClaimFormPage?language=de">German</a>|
<a href="/getTaxClaimFormPage?language=fr">French</a>
</align>
<form:form action="/calculateTax" method="get" modelAttribute="userClaim">
<table>
<tr>
<td id="id1">
<spring:message code="label.employeeId" />
</td>
<td id="id2">
<form:input path="employeeId" id="employeeId" />
</td>
<td id="id3">
<form:errors path="employeeId" />
</td></tr>
<tr>
<td id="id4">
<spring:message code="label.expenseType" />
</td>
<td id="id5">
<form:select path="expenseType" items="${ 
expenseList }" id="expenseType" />
</td>
<td id="id6"></td>
</tr>
<tr>
<td id="id7">
<spring:message code="label.expenseAmount" />
</td>
<td id="id8">
<form:input path="expenseAmt" id="expenseAmount" />
</td>
<td id=id9>
<form:errors path="expenseAmt" />
</td>
</tr>
<tr>
<td><input type="Submit" name="submit" value="Calculate 
Claim" /></td>
<td></td>
</tr>
<tr>
<td><input type="reset" name="reset" value="Clear" /></td>
<td></td>
</tr>
</table>
</form:form>
</body>
</html>
zar
