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

## How I built it

## Challenges I ran into
Unfortunately XSLT-approach has one major performance issue, which is represented on the next picture:
![XSLT-transformation approach performance issue](https://raw.githubusercontent.com/alexay-nesterenko/pega-streaming-framework/master/problem.png "XSLT-transformation approach performance issue")

This problem happens because of:
1.	JDK-embedded free XSLT-processors can only use DOM model as an input for transformation.
2.	To convert Clipboard data model into DOM model we need at first to stream Clipboard page into Clipboard XML, then parse this XML into DOM model, and only after we can apply XSLT-transformation.
3.	These intermediate streaming and parsing steps neglect all the performance benefit, which can be achieved from XSLT-technology usage.

To mitigate this problem, next cornerstone solution was implemented:

Its main idea is:
1.	At first XSLT-transformer Java function “wraps” Clipboard page with DOM interfaces. Particular Clipboard property or page is wrapped «on-the-fly» – only if it is used by the transformation.
2.	And then this function applies compiled XSLT-template to this “wrapped” Clipboard page to, as if it was an ordinary DOM structure.

This way all the intermediate steps with streaming and parsing are eliminated, keeping a performance at the highest possible level. Let’s switch now to the live demo to see, how it works.
## Accomplishments that I'm proud of
As main achievements I consider:
1.	Better performance and lower memory consumption, than with a classic Clipboard-to-XML streaming approach.
2.	XSLT-template represents a single point of modification in the case of transformation changes.
3.	XSLT-transformer function supports both model-first and XSD-first transformation approaches.
4.	Solution proposed fits perfectly into overall Pega concept of reusability and specialization.
## What I learned

## What's next for Advanced XML Mapping Framework for Pega
As next plans I would like to mention:
1.	Implement direct parameterization of XSLT-transformer function by XSD-schema as an alternative to parameterization by XML Stream rule.
2.	XSLT is an open standard, so there are a lot of visual tools available for XSLT-templates editing – both free and commercial. So one of the next tasks can be to integrate some free tools for XSLT-templates visual editing into Pega.

And the last one, and from my prospective the most important future extension. Wrapping of Clipboard with DOM model "opens a door" for application of other XML-based technologies, like XPath or XQuery. For example, next possible usage can be an implementation of reusable Java-function for fast access to specific property value inside complicated Clipboard page with help of just a single XPath expression: <pre><code>.ClientAddress = @xpath("Order/Client[firstName = 'John']/Address/rowdata[.City = 'Munich']/LineAddress")</code></pre>
