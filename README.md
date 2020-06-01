## Inspiration
Classic Pega Clipboard-to-XML streaming approach is represented on the next picture:
![Classic Clipboard-to-XML streaming approach](https://raw.githubusercontent.com/alexay-nesterenko/pega-streaming-framework/master/classic.png "Classic Clipboard-to-XML streaming approach")

It consists of the next steps:
1.	Two types of data models are created: business data model and integration data model.
2.	Clipboard page is transformed from business data model into integration data model with help of Data Transform rules.
3.	 Then Clipboard page is streamed from integration data model into XML with help of XML Stream rules.

Motivation for this project was to improve this streaming process from the next prospective:
1.	Performance. There are 2 processing stages – mapping to integration model and steaming to XML. For big Clibpboard structures 2 stages are giving a performance overhead. Is it possible to have only one processing stage?
2.	Memory consumption. Both Data and Integration model are kept in memory during the processing. Is it possible to have only one data model?
3.	Maintainability. Even to add a new single property, all the layers must be modified – Data Transform rules, Integration model and XML Stream rules. Is it possible to have only one point of modification?
## What it does
To achieve improvements mentioned above, this project applies [Extensible Stylesheet Language Transformations technology (XSLT)](https://www.w3.org/TR/xslt/ "Extensible Stylesheet Language Transformations technology") :
*   XSLT is an XML-based language, which can transform one XML Document Object Model into another one.
*   XSLT has enough expressiveness to perform transformation of any complexity.
*   From performance prospective, JDK-embedded XSLT-processors have an XSLTC extension, which can generate native Java-code from XSLT-template and compile it into executable Java-class "on-the-fly".
*   XSLT-technology fits perfectly into overall Pega concept of reusability and specialization:  one XSLT-template can import all the transformation rules from another one and specialize any of them.
*   Finally, this approach increases maintainability: XSLT-template is doing both transformation and streaming into XML, representing this way a single point of modification in the case of transformation changes.

The solution proposed applies XSLT-technology for efficient Clipboard pages streaming into XML.
## How I built it
For XSLT-templates definition storage inside Pega a special custom Pega rule **Integration-Mapping -> XSLT Template** was developed. This rule enables XSLT-template content editing, as well as its validation with a compilation attempt:
![XSLT Template custom rule](https://raw.githubusercontent.com/alexay-nesterenko/pega-streaming-framework/master/rule.png "XSLT Template custom rule")

Main XSLT-transformation functionality is implemented with help of Java Function, which uses JDK-embedded XSLT-processor for transformation purposes (it supports Oracle JDK, IBM JDK and Open JDK):
<pre><code>@ANUtilities.StreamClipboardToXML(
  myStepPage,
  D_XSLT[Template: "ANOrg-Components-Work.T002_Manual_Basic"].Template,
  null,
  null
)</code></pre>
In the example above this function is called to transform **myStepPage** Clipboard page with help of XSLT-template, defined by castom rule **ANOrg-Components-Work.T002_Manual_Basic**. Access to XSLT-template is implemented through **D_XSLT** node-level Data Page rule. This Data page compiles XSLT-template "on-the-fly" and caches its instance, so it can be reused for multiple transformations later.

XSLT-transformation function implements two execution approaches:
1.	Model-first approach: in this situation application Data Model as well as XSLT-template to stream it to XML are created manually.
2.  XSD-first approach: in this situation XSLT-template can do automatic transformation with help of output format description by [XML Schema Definition (XSD)(https://www.w3.org/TR/xmlschema/ "XML Schema Definition (XSD)").

XSD-first approach is coming from the next question: what if we have already an output XML description in the form of XSD-schema? Is it possible then to simplify XSLT-template development? Yes, it is!

XSD-schema in Pega can be used to generate automatically data model and XML Stream rule. This data model then can be extended during the development with some additional properties, but an output XML format remains strictly defined by XML Stream rule. During the transformation phase this XML Stream rule can be used to automate processing:
![XSD-driven XSLT-transformation approach](https://raw.githubusercontent.com/alexay-nesterenko/pega-streaming-framework/master/schema.png "XSD-driven XSLT-transformation approach")

In this scenario next logic is executed:
1.	XSLT-transformer function is able to read XML Stream rule definition.
2.	And then it can automatically transform a Clipboard page according to this definition, which means – according to XSD-schema.

During this XSD-driven transformation the function is doing multiple things: renaming elements, sorting them, adding required fields and so on. That is an example, how XSLT-transformation function can be called in this situation:
<pre><code>@ANUtilities.StreamClipboardToXML(
  myStepPage,
  D_XSLT[Template: "ANOrg-Components-Work.T002_Manual_Basic"].Template,
  D_Stream[Template: "ANOrg-Components-Work.Order.MapFrom"].Template,
  "sort|skip_default|required|set_default",
)</code></pre>
In this scenario the function receives two additional parameters:
1.	Reference to XML Stream rule to be used. Access to XML Stream rule is implemented through **D_Stream** node-level Data Page rule. This Data page parses XML Stream rule "on-the-fly" and caches it as a **java.util.Map** object instance, so it can be reused for multiple transformations later.
2.	Optionally, a list of additional transformation modes can be provided, which controlls such aspects as elements sorting, preserving required fields, setting default values and so on.
## Challenges I ran into
Unfortunately XSLT-approach has one major performance issue, which is represented on the next picture:
![XSLT-transformation approach performance issue](https://raw.githubusercontent.com/alexay-nesterenko/pega-streaming-framework/master/problem.png "XSLT-transformation approach performance issue")

This problem happens because of:
1.	JDK-embedded free XSLT-processors can only use DOM model as an input for transformation.
2.	To convert Clipboard data model into DOM model we need at first to stream Clipboard page into Clipboard XML, then parse this XML into DOM model, and only after we can apply XSLT-transformation.
3.	These intermediate streaming and parsing steps neglect all the performance benefit, which can be achieved from XSLT-technology usage.

To mitigate this problem, next cornerstone solution was implemented:
![A cornerstone solution to mitigate performance issue](https://raw.githubusercontent.com/alexay-nesterenko/pega-streaming-framework/master/solution.png "A cornerstone solution to mitigate performance issue")

Its main idea is:
1.	At first XSLT-transformer Java function "wraps" Clipboard page with DOM interfaces. Particular Clipboard property or page is wrapped "on-the-fly" – only if it is used by the transformation.
2.	And then this function applies compiled XSLT-template to this "wrapped” Clipboard page to, as if it was an ordinary DOM structure.

This way all the intermediate steps with streaming and parsing are eliminated, keeping a performance at the highest possible level.
## Accomplishments that I'm proud of
As main achievements I consider:
1.	Better performance and lower memory consumption, than with a classic Clipboard-to-XML streaming approach.
2.	XSLT-template represents a single point of modification in the case of transformation changes.
3.	XSLT-transformer function supports both model-first and XSD-first transformation approaches.
4.	Solution proposed fits perfectly into overall Pega concept of reusability and specialization.
## What I learned
test
## What's next for Advanced XML Mapping Framework for Pega
As next plans I would like to mention:
1.	Implement direct parameterization of XSLT-transformer function by XSD-schema as an alternative to parameterization by XML Stream rule.
2.	XSLT is an open standard, so there are a lot of visual tools available for XSLT-templates editing – both free and commercial. So one of the next tasks can be to integrate some free tools for XSLT-templates visual editing into Pega.

And the last one, and from my prospective the most important future extension. Wrapping of Clipboard with DOM model "opens a door" for application of other XML-based technologies, like XPath or XQuery. For example, next possible usage can be an implementation of reusable Java-function for fast access to specific property value inside complicated Clipboard page with help of just a single XPath expression: <pre><code>.ClientAddress = @xpath("Order/Client[firstName = 'John']/Address/rowdata[.City = 'Munich']/LineAddress")</code></pre>
