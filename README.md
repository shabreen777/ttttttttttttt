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

