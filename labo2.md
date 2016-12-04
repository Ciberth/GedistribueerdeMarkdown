# Handlers

3 Projecten:
    - EquationService
    - TestEquationService
    - TestWoordenboekService

## EquationService

(In package ws)

### EquationService_handler.xml:

```xml

<?xml version="1.0" encoding="UTF-8"?>
<handler-chains xmlns="http://java.sun.com/xml/ns/javaee">
  <handler-chain>
    <handler>
      <handler-name>ws.TestLogicalHandler</handler-name>
      <handler-class>ws.TestLogicalHandler</handler-class>
    </handler>
<!--    <handler>
      <handler-name>ws.TestMessageHandler</handler-name>
      <handler-class>ws.TestMessageHandler</handler-class>
    </handler>-->
  </handler-chain>
</handler-chains>


```

### EquationService.java

```java

```

```java

@WebService(serviceName = "EquationService")
@HandlerChain(file = "EquationService_handler.xml")
public class EquationService {

    @WebMethod(operationName = "hello")
    public String hello(@WebParam(name = "name") String txt) {
        return "Hello " + txt + " !";
    }
    
    @WebMethod(operationName = "bepaalNulpunten")
    public double[] solveQuadratic(@WebParam(name = "c1") double a, @WebParam(name = "c2") double b, @WebParam(name = "c3") double c) {
         // solve a x^2 + b x + c
        double discr = b * b - 4 * a * c;
        if (discr < 0) {
            return new double[0];
        } else if (Math.abs(discr) < 1e-10) {
            return new double[]{(-b + Math.sqrt(discr)) / (2 * a)};
        } else {
            return new double[]{
                (-b + Math.sqrt(discr)) / (2 * a),
                (-b - Math.sqrt(discr)) / (2 * a)
            };
        }
    }
}


```

### TestLogicalHandler.java

```java
public class TestLogicalHandler implements LogicalHandler<LogicalMessageContext> {

    @Override
    public boolean handleMessage(LogicalMessageContext messageContext) {
        Boolean outbound = (Boolean) messageContext.get(MessageContext.MESSAGE_OUTBOUND_PROPERTY);
        if (!outbound) {
            try {
                LogicalMessage msg = messageContext.getMessage();
                Source inhoud = msg.getPayload();
                TransformerFactory factory = TransformerFactory.newInstance();
                Transformer tf = factory.newTransformer(new StreamSource(this.getClass().getClassLoader().getResourceAsStream("ws/nieuweVersie.xsl")));
                DocumentBuilderFactory dFactory = DocumentBuilderFactory.newInstance();
                DocumentBuilder builder = dFactory.newDocumentBuilder();
                Document document = builder.newDocument();
                DOMResult aangepast = new DOMResult(document);
                tf.transform(inhoud, aangepast);
                msg.setPayload(new DOMSource(aangepast.getNode()));
            } catch (TransformerConfigurationException | ParserConfigurationException ex) {
                Logger.getLogger(TestLogicalHandler.class.getName()).log(Level.SEVERE, null, ex);
            } catch (TransformerException ex) {
                Logger.getLogger(TestLogicalHandler.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
        return true;
    }

    @Override
    public boolean handleFault(LogicalMessageContext messageContext) {
        return true;
    }

    @Override
    public void close(MessageContext context) {
    }

}

```

### TestMessageHandler.java

```java

public class TestMessageHandler implements SOAPHandler<SOAPMessageContext> {

    @Override
    public boolean handleMessage(SOAPMessageContext messageContext) {
        /*message direction, true for outbound messages, false for inbound*/
        Boolean outbound = (Boolean) messageContext.get(MessageContext.MESSAGE_OUTBOUND_PROPERTY);
        if (!outbound) {
            /*if the message is an outband message then get its context*/
            SOAPMessage msg = messageContext.getMessage();
            /*start handling the msg body <<<it is xml, SO handling nodes>>>*/
            try {
                SOAPBody body = msg.getSOAPBody();
                Node bepaalNulpunten = body.getFirstChild();
                NodeList coefficienten = bepaalNulpunten.getChildNodes();
                for (int i = 0; i < coefficienten.getLength(); i++) {
                    Node coefficient = coefficienten.item(i);
                    switch (coefficient.getLocalName()) {
                        case "a":
                            veranderNodeName("c0", coefficient, bepaalNulpunten);
                            break;
                        case "b":
                            veranderNodeName("c1", coefficient, bepaalNulpunten);
                            break;
                        case "c":
                            veranderNodeName("c2", coefficient, bepaalNulpunten);
                            break;
                    }
                }
                msg.saveChanges();
            } catch (SOAPException ex) {
                Logger.getLogger(TestMessageHandler.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
        return true;
    }

    private void veranderNodeName(String naam, Node coefficient, Node bepaalNulpunten) throws DOMException {
        Document document = coefficient.getOwnerDocument();
        Node nieuw = document.createElement(naam);
        nieuw.appendChild(document.createTextNode(coefficient.getFirstChild().getNodeValue()));
        bepaalNulpunten.replaceChild(nieuw, coefficient);
    }

    @Override
    public Set<QName> getHeaders() {
        return Collections.EMPTY_SET;
    }

    @Override
    public boolean handleFault(SOAPMessageContext messageContext) {
        return true;
    }

    @Override
    public void close(MessageContext context) {
    }

}


```

## Project TestEquationService 

### TestEquationService.java

```java
public class TestEquationService {

    /**
     * @param args the command line arguments
     */
    public static void main(String[] args) {

        for (double root : berekenNulpunten(1, -5, 6)) {
            System.out.println(root);
        }
        
        for (double root : berekenNulpunten(2, -4, 2)) {
            System.out.println(root);
        }
        
        for (double root : berekenNulpunten(5, 1, 2)) {
            System.out.println(root);
        }
    }
/*This class is created automatically and the class "EquationService_Service" is the implemented class of the interface "EquationService" */
    /*If the class "EquationService" is used to create an instanse (object) of the equationservice then an exception of an abstract class is showed*/
    private static java.util.List<java.lang.Double> berekenNulpunten(double a, double b, double c) {
        EquationService_Service service = new EquationService_Service();
        EquationService port = service.getEquationServicePort();
        return port.bepaalNulpunten(a, b, c);
    }
    
}
```

### EquationService.wsdl

```xml

<?xml version='1.0' encoding='UTF-8'?><!-- Published by JAX-WS RI (http://jax-ws.java.net). RI's version is Metro/2.3.1-b419 (branches/2.3.1.x-7937; 2014-08-04T08:11:03+0000) JAXWS-RI/2.2.10-b140803.1500 JAXWS-API/2.2.11 JAXB-RI/2.2.10-b140802.1033 JAXB-API/2.2.12-b140109.1041 svn-revision#unknown. --><!-- Generated by JAX-WS RI (http://jax-ws.java.net). RI's version is Metro/2.3.1-b419 (branches/2.3.1.x-7937; 2014-08-04T08:11:03+0000) JAXWS-RI/2.2.10-b140803.1500 JAXWS-API/2.2.11 JAXB-RI/2.2.10-b140802.1033 JAXB-API/2.2.12-b140109.1041 svn-revision#unknown. --><definitions xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd" xmlns:wsp="http://www.w3.org/ns/ws-policy" xmlns:wsp1_2="http://schemas.xmlsoap.org/ws/2004/09/policy" xmlns:wsam="http://www.w3.org/2007/05/addressing/metadata" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:tns="http://ws/" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns="http://schemas.xmlsoap.org/wsdl/" targetNamespace="http://ws/" name="EquationService">
<types>
  <xsd:schema>
    <xsd:import namespace="http://ws/" schemaLocation="http://localhost:8080/EquationService/EquationService?xsd=1"/>
  </xsd:schema>
</types>
<message name="bepaalNulpunten">
  <part name="parameters" element="tns:bepaalNulpunten"/>
</message>
<message name="bepaalNulpuntenResponse">
  <part name="parameters" element="tns:bepaalNulpuntenResponse"/>
</message>
<message name="hello">
  <part name="parameters" element="tns:hello"/>
</message>
<message name="helloResponse">
  <part name="parameters" element="tns:helloResponse"/>
</message>
<portType name="EquationService">
  <operation name="bepaalNulpunten">
    <input wsam:Action="http://ws/EquationService/bepaalNulpuntenRequest" message="tns:bepaalNulpunten"/>
    <output wsam:Action="http://ws/EquationService/bepaalNulpuntenResponse" message="tns:bepaalNulpuntenResponse"/>
  </operation>
  <operation name="hello">
    <input wsam:Action="http://ws/EquationService/helloRequest" message="tns:hello"/>
    <output wsam:Action="http://ws/EquationService/helloResponse" message="tns:helloResponse"/>
  </operation>
</portType>
<binding name="EquationServicePortBinding" type="tns:EquationService">
  <soap:binding transport="http://schemas.xmlsoap.org/soap/http" style="document"/>
  <operation name="bepaalNulpunten">
    <soap:operation soapAction=""/>
    <input>
      <soap:body use="literal"/>
    </input>
    <output>
      <soap:body use="literal"/>
    </output>
  </operation>
  <operation name="hello">
    <soap:operation soapAction=""/>
    <input>
      <soap:body use="literal"/>
    </input>
    <output>
      <soap:body use="literal"/>
    </output>
  </operation>
</binding>
<service name="EquationService">
  <port name="EquationServicePort" binding="tns:EquationServicePortBinding">
    <soap:address location="http://localhost:8080/EquationService/EquationService"/>
  </port>
</service>
</definitions>

```

### EquationService.xsd_1.xsd

```xml

<?xml version='1.0' encoding='UTF-8'?><!-- Published by JAX-WS RI (http://jax-ws.java.net). RI's version is Metro/2.3.1-b419 (branches/2.3.1.x-7937; 2014-08-04T08:11:03+0000) JAXWS-RI/2.2.10-b140803.1500 JAXWS-API/2.2.11 JAXB-RI/2.2.10-b140802.1033 JAXB-API/2.2.12-b140109.1041 svn-revision#unknown. --><xs:schema xmlns:tns="http://ws/" xmlns:xs="http://www.w3.org/2001/XMLSchema" version="1.0" targetNamespace="http://ws/">

<xs:element name="bepaalNulpunten" type="tns:bepaalNulpunten"/>

<xs:element name="bepaalNulpuntenResponse" type="tns:bepaalNulpuntenResponse"/>

<xs:element name="hello" type="tns:hello"/>

<xs:element name="helloResponse" type="tns:helloResponse"/>

<xs:complexType name="hello">
  <xs:sequence>
    <xs:element name="name" type="xs:string" minOccurs="0"/>
  </xs:sequence>
</xs:complexType>

<xs:complexType name="helloResponse">
  <xs:sequence>
    <xs:element name="return" type="xs:string" minOccurs="0"/>
  </xs:sequence>
</xs:complexType>

<xs:complexType name="bepaalNulpunten">
  <xs:sequence>
    <xs:element name="a" type="xs:double"/>
    <xs:element name="b" type="xs:double"/>
    <xs:element name="c" type="xs:double"/>
  </xs:sequence>
</xs:complexType>

<xs:complexType name="bepaalNulpuntenResponse">
  <xs:sequence>
    <xs:element name="return" type="xs:double" nillable="true" minOccurs="0" maxOccurs="unbounded"/>
  </xs:sequence>
</xs:complexType>
</xs:schema>

```

## Project TestWoordenboekService

### DictService.asmx.wsdl

```xml

<?xml version="1.0" encoding="utf-8"?>
<wsdl:definitions xmlns:s="http://www.w3.org/2001/XMLSchema" xmlns:soap12="http://schemas.xmlsoap.org/wsdl/soap12/" xmlns:mime="http://schemas.xmlsoap.org/wsdl/mime/" xmlns:tns="http://services.aonaware.com/webservices/" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:tm="http://microsoft.com/wsdl/mime/textMatching/" xmlns:http="http://schemas.xmlsoap.org/wsdl/http/" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" targetNamespace="http://services.aonaware.com/webservices/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">
  <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Word Dictionary Web Service</wsdl:documentation>
  <wsdl:types>
    <s:schema elementFormDefault="qualified" targetNamespace="http://services.aonaware.com/webservices/">
      <s:element name="ServerInfo">
        <s:complexType />
      </s:element>
      <s:element name="ServerInfoResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="ServerInfoResult" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="DictionaryList">
        <s:complexType />
      </s:element>
      <s:element name="DictionaryListResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="DictionaryListResult" type="tns:ArrayOfDictionary" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:complexType name="ArrayOfDictionary">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="unbounded" name="Dictionary" nillable="true" type="tns:Dictionary" />
        </s:sequence>
      </s:complexType>
      <s:complexType name="Dictionary">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="1" name="Id" type="s:string" />
          <s:element minOccurs="0" maxOccurs="1" name="Name" type="s:string" />
        </s:sequence>
      </s:complexType>
      <s:element name="DictionaryListExtended">
        <s:complexType />
      </s:element>
      <s:element name="DictionaryListExtendedResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="DictionaryListExtendedResult" type="tns:ArrayOfDictionary" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="DictionaryInfo">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="dictId" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="DictionaryInfoResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="DictionaryInfoResult" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="Define">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="word" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="DefineResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="DefineResult" type="tns:WordDefinition" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:complexType name="WordDefinition">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="1" name="Word" type="s:string" />
          <s:element minOccurs="0" maxOccurs="1" name="Definitions" type="tns:ArrayOfDefinition" />
        </s:sequence>
      </s:complexType>
      <s:complexType name="ArrayOfDefinition">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="unbounded" name="Definition" nillable="true" type="tns:Definition" />
        </s:sequence>
      </s:complexType>
      <s:complexType name="Definition">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="1" name="Word" type="s:string" />
          <s:element minOccurs="0" maxOccurs="1" name="Dictionary" type="tns:Dictionary" />
          <s:element minOccurs="0" maxOccurs="1" name="WordDefinition" type="s:string" />
        </s:sequence>
      </s:complexType>
      <s:element name="DefineInDict">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="dictId" type="s:string" />
            <s:element minOccurs="0" maxOccurs="1" name="word" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="DefineInDictResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="DefineInDictResult" type="tns:WordDefinition" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="StrategyList">
        <s:complexType />
      </s:element>
      <s:element name="StrategyListResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="StrategyListResult" type="tns:ArrayOfStrategy" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:complexType name="ArrayOfStrategy">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="unbounded" name="Strategy" nillable="true" type="tns:Strategy" />
        </s:sequence>
      </s:complexType>
      <s:complexType name="Strategy">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="1" name="Id" type="s:string" />
          <s:element minOccurs="0" maxOccurs="1" name="Description" type="s:string" />
        </s:sequence>
      </s:complexType>
      <s:element name="Match">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="word" type="s:string" />
            <s:element minOccurs="0" maxOccurs="1" name="strategy" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="MatchResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="MatchResult" type="tns:ArrayOfDictionaryWord" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:complexType name="ArrayOfDictionaryWord">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="unbounded" name="DictionaryWord" nillable="true" type="tns:DictionaryWord" />
        </s:sequence>
      </s:complexType>
      <s:complexType name="DictionaryWord">
        <s:sequence>
          <s:element minOccurs="0" maxOccurs="1" name="DictionaryId" type="s:string" />
          <s:element minOccurs="0" maxOccurs="1" name="Word" type="s:string" />
        </s:sequence>
      </s:complexType>
      <s:element name="MatchInDict">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="dictId" type="s:string" />
            <s:element minOccurs="0" maxOccurs="1" name="word" type="s:string" />
            <s:element minOccurs="0" maxOccurs="1" name="strategy" type="s:string" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="MatchInDictResponse">
        <s:complexType>
          <s:sequence>
            <s:element minOccurs="0" maxOccurs="1" name="MatchInDictResult" type="tns:ArrayOfDictionaryWord" />
          </s:sequence>
        </s:complexType>
      </s:element>
      <s:element name="string" nillable="true" type="s:string" />
      <s:element name="ArrayOfDictionary" nillable="true" type="tns:ArrayOfDictionary" />
      <s:element name="WordDefinition" nillable="true" type="tns:WordDefinition" />
      <s:element name="ArrayOfStrategy" nillable="true" type="tns:ArrayOfStrategy" />
      <s:element name="ArrayOfDictionaryWord" nillable="true" type="tns:ArrayOfDictionaryWord" />
    </s:schema>
  </wsdl:types>
  <wsdl:message name="ServerInfoSoapIn">
    <wsdl:part name="parameters" element="tns:ServerInfo" />
  </wsdl:message>
  <wsdl:message name="ServerInfoSoapOut">
    <wsdl:part name="parameters" element="tns:ServerInfoResponse" />
  </wsdl:message>
  <wsdl:message name="DictionaryListSoapIn">
    <wsdl:part name="parameters" element="tns:DictionaryList" />
  </wsdl:message>
  <wsdl:message name="DictionaryListSoapOut">
    <wsdl:part name="parameters" element="tns:DictionaryListResponse" />
  </wsdl:message>
  <wsdl:message name="DictionaryListExtendedSoapIn">
    <wsdl:part name="parameters" element="tns:DictionaryListExtended" />
  </wsdl:message>
  <wsdl:message name="DictionaryListExtendedSoapOut">
    <wsdl:part name="parameters" element="tns:DictionaryListExtendedResponse" />
  </wsdl:message>
  <wsdl:message name="DictionaryInfoSoapIn">
    <wsdl:part name="parameters" element="tns:DictionaryInfo" />
  </wsdl:message>
  <wsdl:message name="DictionaryInfoSoapOut">
    <wsdl:part name="parameters" element="tns:DictionaryInfoResponse" />
  </wsdl:message>
  <wsdl:message name="DefineSoapIn">
    <wsdl:part name="parameters" element="tns:Define" />
  </wsdl:message>
  <wsdl:message name="DefineSoapOut">
    <wsdl:part name="parameters" element="tns:DefineResponse" />
  </wsdl:message>
  <wsdl:message name="DefineInDictSoapIn">
    <wsdl:part name="parameters" element="tns:DefineInDict" />
  </wsdl:message>
  <wsdl:message name="DefineInDictSoapOut">
    <wsdl:part name="parameters" element="tns:DefineInDictResponse" />
  </wsdl:message>
  <wsdl:message name="StrategyListSoapIn">
    <wsdl:part name="parameters" element="tns:StrategyList" />
  </wsdl:message>
  <wsdl:message name="StrategyListSoapOut">
    <wsdl:part name="parameters" element="tns:StrategyListResponse" />
  </wsdl:message>
  <wsdl:message name="MatchSoapIn">
    <wsdl:part name="parameters" element="tns:Match" />
  </wsdl:message>
  <wsdl:message name="MatchSoapOut">
    <wsdl:part name="parameters" element="tns:MatchResponse" />
  </wsdl:message>
  <wsdl:message name="MatchInDictSoapIn">
    <wsdl:part name="parameters" element="tns:MatchInDict" />
  </wsdl:message>
  <wsdl:message name="MatchInDictSoapOut">
    <wsdl:part name="parameters" element="tns:MatchInDictResponse" />
  </wsdl:message>
  <wsdl:message name="ServerInfoHttpGetIn" />
  <wsdl:message name="ServerInfoHttpGetOut">
    <wsdl:part name="Body" element="tns:string" />
  </wsdl:message>
  <wsdl:message name="DictionaryListHttpGetIn" />
  <wsdl:message name="DictionaryListHttpGetOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionary" />
  </wsdl:message>
  <wsdl:message name="DictionaryListExtendedHttpGetIn" />
  <wsdl:message name="DictionaryListExtendedHttpGetOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionary" />
  </wsdl:message>
  <wsdl:message name="DictionaryInfoHttpGetIn">
    <wsdl:part name="dictId" type="s:string" />
  </wsdl:message>
  <wsdl:message name="DictionaryInfoHttpGetOut">
    <wsdl:part name="Body" element="tns:string" />
  </wsdl:message>
  <wsdl:message name="DefineHttpGetIn">
    <wsdl:part name="word" type="s:string" />
  </wsdl:message>
  <wsdl:message name="DefineHttpGetOut">
    <wsdl:part name="Body" element="tns:WordDefinition" />
  </wsdl:message>
  <wsdl:message name="DefineInDictHttpGetIn">
    <wsdl:part name="dictId" type="s:string" />
    <wsdl:part name="word" type="s:string" />
  </wsdl:message>
  <wsdl:message name="DefineInDictHttpGetOut">
    <wsdl:part name="Body" element="tns:WordDefinition" />
  </wsdl:message>
  <wsdl:message name="StrategyListHttpGetIn" />
  <wsdl:message name="StrategyListHttpGetOut">
    <wsdl:part name="Body" element="tns:ArrayOfStrategy" />
  </wsdl:message>
  <wsdl:message name="MatchHttpGetIn">
    <wsdl:part name="word" type="s:string" />
    <wsdl:part name="strategy" type="s:string" />
  </wsdl:message>
  <wsdl:message name="MatchHttpGetOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionaryWord" />
  </wsdl:message>
  <wsdl:message name="MatchInDictHttpGetIn">
    <wsdl:part name="dictId" type="s:string" />
    <wsdl:part name="word" type="s:string" />
    <wsdl:part name="strategy" type="s:string" />
  </wsdl:message>
  <wsdl:message name="MatchInDictHttpGetOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionaryWord" />
  </wsdl:message>
  <wsdl:message name="ServerInfoHttpPostIn" />
  <wsdl:message name="ServerInfoHttpPostOut">
    <wsdl:part name="Body" element="tns:string" />
  </wsdl:message>
  <wsdl:message name="DictionaryListHttpPostIn" />
  <wsdl:message name="DictionaryListHttpPostOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionary" />
  </wsdl:message>
  <wsdl:message name="DictionaryListExtendedHttpPostIn" />
  <wsdl:message name="DictionaryListExtendedHttpPostOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionary" />
  </wsdl:message>
  <wsdl:message name="DictionaryInfoHttpPostIn">
    <wsdl:part name="dictId" type="s:string" />
  </wsdl:message>
  <wsdl:message name="DictionaryInfoHttpPostOut">
    <wsdl:part name="Body" element="tns:string" />
  </wsdl:message>
  <wsdl:message name="DefineHttpPostIn">
    <wsdl:part name="word" type="s:string" />
  </wsdl:message>
  <wsdl:message name="DefineHttpPostOut">
    <wsdl:part name="Body" element="tns:WordDefinition" />
  </wsdl:message>
  <wsdl:message name="DefineInDictHttpPostIn">
    <wsdl:part name="dictId" type="s:string" />
    <wsdl:part name="word" type="s:string" />
  </wsdl:message>
  <wsdl:message name="DefineInDictHttpPostOut">
    <wsdl:part name="Body" element="tns:WordDefinition" />
  </wsdl:message>
  <wsdl:message name="StrategyListHttpPostIn" />
  <wsdl:message name="StrategyListHttpPostOut">
    <wsdl:part name="Body" element="tns:ArrayOfStrategy" />
  </wsdl:message>
  <wsdl:message name="MatchHttpPostIn">
    <wsdl:part name="word" type="s:string" />
    <wsdl:part name="strategy" type="s:string" />
  </wsdl:message>
  <wsdl:message name="MatchHttpPostOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionaryWord" />
  </wsdl:message>
  <wsdl:message name="MatchInDictHttpPostIn">
    <wsdl:part name="dictId" type="s:string" />
    <wsdl:part name="word" type="s:string" />
    <wsdl:part name="strategy" type="s:string" />
  </wsdl:message>
  <wsdl:message name="MatchInDictHttpPostOut">
    <wsdl:part name="Body" element="tns:ArrayOfDictionaryWord" />
  </wsdl:message>
  <wsdl:portType name="DictServiceSoap">
    <wsdl:operation name="ServerInfo">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Show remote server information</wsdl:documentation>
      <wsdl:input message="tns:ServerInfoSoapIn" />
      <wsdl:output message="tns:ServerInfoSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Returns a list of available dictionaries</wsdl:documentation>
      <wsdl:input message="tns:DictionaryListSoapIn" />
      <wsdl:output message="tns:DictionaryListSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Returns a list of advanced dictionaries (e.g. translating dictionaries)</wsdl:documentation>
      <wsdl:input message="tns:DictionaryListExtendedSoapIn" />
      <wsdl:output message="tns:DictionaryListExtendedSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Show information about the specified dictionary</wsdl:documentation>
      <wsdl:input message="tns:DictionaryInfoSoapIn" />
      <wsdl:output message="tns:DictionaryInfoSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="Define">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Define given word, returning definitions from all dictionaries</wsdl:documentation>
      <wsdl:input message="tns:DefineSoapIn" />
      <wsdl:output message="tns:DefineSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Define given word, returning definitions from specified dictionary</wsdl:documentation>
      <wsdl:input message="tns:DefineInDictSoapIn" />
      <wsdl:output message="tns:DefineInDictSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Return list of all available strategies on the server</wsdl:documentation>
      <wsdl:input message="tns:StrategyListSoapIn" />
      <wsdl:output message="tns:StrategyListSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="Match">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Look for matching words in all dictionaries using the given strategy</wsdl:documentation>
      <wsdl:input message="tns:MatchSoapIn" />
      <wsdl:output message="tns:MatchSoapOut" />
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Look for matching words in the specified dictionary using the given strategy</wsdl:documentation>
      <wsdl:input message="tns:MatchInDictSoapIn" />
      <wsdl:output message="tns:MatchInDictSoapOut" />
    </wsdl:operation>
  </wsdl:portType>
  <wsdl:portType name="DictServiceHttpGet">
    <wsdl:operation name="ServerInfo">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Show remote server information</wsdl:documentation>
      <wsdl:input message="tns:ServerInfoHttpGetIn" />
      <wsdl:output message="tns:ServerInfoHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Returns a list of available dictionaries</wsdl:documentation>
      <wsdl:input message="tns:DictionaryListHttpGetIn" />
      <wsdl:output message="tns:DictionaryListHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Returns a list of advanced dictionaries (e.g. translating dictionaries)</wsdl:documentation>
      <wsdl:input message="tns:DictionaryListExtendedHttpGetIn" />
      <wsdl:output message="tns:DictionaryListExtendedHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Show information about the specified dictionary</wsdl:documentation>
      <wsdl:input message="tns:DictionaryInfoHttpGetIn" />
      <wsdl:output message="tns:DictionaryInfoHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="Define">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Define given word, returning definitions from all dictionaries</wsdl:documentation>
      <wsdl:input message="tns:DefineHttpGetIn" />
      <wsdl:output message="tns:DefineHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Define given word, returning definitions from specified dictionary</wsdl:documentation>
      <wsdl:input message="tns:DefineInDictHttpGetIn" />
      <wsdl:output message="tns:DefineInDictHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Return list of all available strategies on the server</wsdl:documentation>
      <wsdl:input message="tns:StrategyListHttpGetIn" />
      <wsdl:output message="tns:StrategyListHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="Match">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Look for matching words in all dictionaries using the given strategy</wsdl:documentation>
      <wsdl:input message="tns:MatchHttpGetIn" />
      <wsdl:output message="tns:MatchHttpGetOut" />
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Look for matching words in the specified dictionary using the given strategy</wsdl:documentation>
      <wsdl:input message="tns:MatchInDictHttpGetIn" />
      <wsdl:output message="tns:MatchInDictHttpGetOut" />
    </wsdl:operation>
  </wsdl:portType>
  <wsdl:portType name="DictServiceHttpPost">
    <wsdl:operation name="ServerInfo">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Show remote server information</wsdl:documentation>
      <wsdl:input message="tns:ServerInfoHttpPostIn" />
      <wsdl:output message="tns:ServerInfoHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Returns a list of available dictionaries</wsdl:documentation>
      <wsdl:input message="tns:DictionaryListHttpPostIn" />
      <wsdl:output message="tns:DictionaryListHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Returns a list of advanced dictionaries (e.g. translating dictionaries)</wsdl:documentation>
      <wsdl:input message="tns:DictionaryListExtendedHttpPostIn" />
      <wsdl:output message="tns:DictionaryListExtendedHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Show information about the specified dictionary</wsdl:documentation>
      <wsdl:input message="tns:DictionaryInfoHttpPostIn" />
      <wsdl:output message="tns:DictionaryInfoHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="Define">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Define given word, returning definitions from all dictionaries</wsdl:documentation>
      <wsdl:input message="tns:DefineHttpPostIn" />
      <wsdl:output message="tns:DefineHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Define given word, returning definitions from specified dictionary</wsdl:documentation>
      <wsdl:input message="tns:DefineInDictHttpPostIn" />
      <wsdl:output message="tns:DefineInDictHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Return list of all available strategies on the server</wsdl:documentation>
      <wsdl:input message="tns:StrategyListHttpPostIn" />
      <wsdl:output message="tns:StrategyListHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="Match">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Look for matching words in all dictionaries using the given strategy</wsdl:documentation>
      <wsdl:input message="tns:MatchHttpPostIn" />
      <wsdl:output message="tns:MatchHttpPostOut" />
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Look for matching words in the specified dictionary using the given strategy</wsdl:documentation>
      <wsdl:input message="tns:MatchInDictHttpPostIn" />
      <wsdl:output message="tns:MatchInDictHttpPostOut" />
    </wsdl:operation>
  </wsdl:portType>
  <wsdl:binding name="DictServiceSoap" type="tns:DictServiceSoap">
    <soap:binding transport="http://schemas.xmlsoap.org/soap/http" />
    <wsdl:operation name="ServerInfo">
      <soap:operation soapAction="http://services.aonaware.com/webservices/ServerInfo" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <soap:operation soapAction="http://services.aonaware.com/webservices/DictionaryList" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <soap:operation soapAction="http://services.aonaware.com/webservices/DictionaryListExtended" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <soap:operation soapAction="http://services.aonaware.com/webservices/DictionaryInfo" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Define">
      <soap:operation soapAction="http://services.aonaware.com/webservices/Define" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <soap:operation soapAction="http://services.aonaware.com/webservices/DefineInDict" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <soap:operation soapAction="http://services.aonaware.com/webservices/StrategyList" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Match">
      <soap:operation soapAction="http://services.aonaware.com/webservices/Match" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <soap:operation soapAction="http://services.aonaware.com/webservices/MatchInDict" style="document" />
      <wsdl:input>
        <soap:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
  </wsdl:binding>
  <wsdl:binding name="DictServiceSoap12" type="tns:DictServiceSoap">
    <soap12:binding transport="http://schemas.xmlsoap.org/soap/http" />
    <wsdl:operation name="ServerInfo">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/ServerInfo" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/DictionaryList" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/DictionaryListExtended" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/DictionaryInfo" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Define">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/Define" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/DefineInDict" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/StrategyList" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Match">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/Match" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <soap12:operation soapAction="http://services.aonaware.com/webservices/MatchInDict" style="document" />
      <wsdl:input>
        <soap12:body use="literal" />
      </wsdl:input>
      <wsdl:output>
        <soap12:body use="literal" />
      </wsdl:output>
    </wsdl:operation>
  </wsdl:binding>
  <wsdl:binding name="DictServiceHttpGet" type="tns:DictServiceHttpGet">
    <http:binding verb="GET" />
    <wsdl:operation name="ServerInfo">
      <http:operation location="/ServerInfo" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <http:operation location="/DictionaryList" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <http:operation location="/DictionaryListExtended" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <http:operation location="/DictionaryInfo" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Define">
      <http:operation location="/Define" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <http:operation location="/DefineInDict" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <http:operation location="/StrategyList" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Match">
      <http:operation location="/Match" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <http:operation location="/MatchInDict" />
      <wsdl:input>
        <http:urlEncoded />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
  </wsdl:binding>
  <wsdl:binding name="DictServiceHttpPost" type="tns:DictServiceHttpPost">
    <http:binding verb="POST" />
    <wsdl:operation name="ServerInfo">
      <http:operation location="/ServerInfo" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryList">
      <http:operation location="/DictionaryList" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryListExtended">
      <http:operation location="/DictionaryListExtended" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DictionaryInfo">
      <http:operation location="/DictionaryInfo" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Define">
      <http:operation location="/Define" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="DefineInDict">
      <http:operation location="/DefineInDict" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="StrategyList">
      <http:operation location="/StrategyList" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="Match">
      <http:operation location="/Match" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
    <wsdl:operation name="MatchInDict">
      <http:operation location="/MatchInDict" />
      <wsdl:input>
        <mime:content type="application/x-www-form-urlencoded" />
      </wsdl:input>
      <wsdl:output>
        <mime:mimeXml part="Body" />
      </wsdl:output>
    </wsdl:operation>
  </wsdl:binding>
  <wsdl:service name="DictService">
    <wsdl:documentation xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/">Word Dictionary Web Service</wsdl:documentation>
    <wsdl:port name="DictServiceSoap" binding="tns:DictServiceSoap">
      <soap:address location="http://services.aonaware.com/DictService/DictService.asmx" />
    </wsdl:port>
    <wsdl:port name="DictServiceSoap12" binding="tns:DictServiceSoap12">
      <soap12:address location="http://services.aonaware.com/DictService/DictService.asmx" />
    </wsdl:port>
    <wsdl:port name="DictServiceHttpGet" binding="tns:DictServiceHttpGet">
      <http:address location="http://services.aonaware.com/DictService/DictService.asmx" />
    </wsdl:port>
    <wsdl:port name="DictServiceHttpPost" binding="tns:DictServiceHttpPost">
      <http:address location="http://services.aonaware.com/DictService/DictService.asmx" />
    </wsdl:port>
  </wsdl:service>
</wsdl:definitions>

```

Rest zie vorig labo!