SendStatuses
============

EDI (API) Documentation for the Standard, XML-Based, Vendor Status HTTP Endpoint

Overview
----------

Pilot Freight Services can receive specially-formatted XML documents containing freight status information from our suppliers. This information can be transferred to our HTTP endpoint or put into a file which can be transferred via FTP-- either downloaded by us from an FTP server hosted by the supplier or uploaded to an FTP server hosted by us.

At this time, only primary, transportation-related events can be submitted using this system. Examples include freight pickup, staging for delivery, and delivery. Calls made to recipients for scheduling-- and the agreed-upon schedule-- cannot be transmitted using this service at this time. Those improvements are on the road map.

See also our [GetManifests](https://github.com/MFSTech/GetManifests) documentation for additional context.

If you have any questions, please email PFD.EDI@PilotDelivers.com.

Example Files
-------------

* [Sample Inbound Transmission](StatusXMLInbound-Sample.xml) (From You to Us)
* Sample Response (From Us to You)
 * [Success](StatusXMLInbound-Response-Success-Sample.xml)
 * [Failure](StatusXMLInbound-Response-Failure-Sample.xml)

Business Concepts
-----------------

We contract with suppliers to move freight and perform services, and we expect to be provided information as to the progress of that movement and performance. We send information about the work to be performed using our [outbound Manifest transmission system](https://github.com/MFSTech/GetManifests) and we are prepared to receive the progress information using this inbound Manifest status system.

### Manifests vs Shipments

Our customers request us to move their shipments, which often consist of large consumer goods, usually from their warehouses to residences. We consolidate those shipments together into manifests, which form the basis of our engagements with suppliers.

For example, a customer might ask us to ship 30 pieces to 20 different locations. That will be split up into 20 different shipments. We will ask a supplier to pick up the 30 pieces from a customer's warehouse. That pickup is a single manifest and we expect to be charged based on the aggregate weight of the full group. To help with reconciliation at origin, we will also provide the individual shipment information, but the official request is at the manifest level, and we expect statuses back at the manifest level.

After the pickup is complete, shipments are regrouped into different manifests for long distance travel, typically called a line haul. In the air freight context, these would be called Master AirWay Bills (MAWBs), and that's a term we still use for these manifests. So from our example 30 pieces going to 20 different locations, there might be 10 different groups going to 10 different markets. Each of those groups of one or more shipments would become a new manifest.

Once the long distance manifests reach their destination markets, they are split up into 20 different manifests for delivery to the 20 different final destinations.

So a single shipment from our customer usually is involved in at least three manifests. Similarly, most pickup and long distance manifests have more than one shipment. However, most deliveries, because they are to individual residences, have only one shipment.

### Statuses vs Other Activity

This system is designed to receive statuses on freight movement. Typical freight statuses include concepts such as the product being picked up from the origin location, arriving at and departing from transportation terminals, and finally being delivered to the destination location.

As the services our customers ask us to perform have become more complex, we now do more than just move freight. For example, we often perform "deluxing" services for our furniture customers (which involves unpacking and reviewing products for damage). Sometimes, those activities require progress information which is unrelated to actual freight movement, but those new types of events have been integrated into our main status system.

However, there are some services we ask our suppliers to perform which we track outside of normal statuses. Examples include the calls suppliers make to schedule deliveries. At this time, we cannot map to our statuses any information received about those calls through this system. Similarly, many of our suppliers will coordinate a delivery date with our customers. We have no provision to receive that scheduling information in a structured manner using this system. Suppliers who are responsible for these services must continue to use other tools (e.g. our supplier-facing web application [Clarity](http://www.mfsclarity.com/Clarity2)) to inform us of their progress, officially.

That said, we built this status receiving system to log everything it receives, even if we are not able to map the information to a particular status event. Therefore, we encourage suppliers to send to us all of the activity information they have. We will pull out the bits we can use programmatically, but all of it will be made available to our operations staff to assist them in understanding what is occurring with a manifest.

Content
-------

The basic format is as follows:

* `StatusEntry`
    * `TransmissionID`
    * `TransmissionGenerated`
    * `TransmissionGeneratedUTC`
    * `Authentication`
    * `StatusSubmissions`
        * `StatusSubmission`
            * `SubmissionID`
            * `Shipment`
                 * `KeyDates`
                 * `ControlNumbers`
                 * `Pieces`
                 * `Charges`
                 * `ShipmentSummary`
            * `Status`

To accept status information, from a technical perspective, at a bare minimum we require:

* Valid credentials
* Manifest identification.
* Status type (vendor or ours).
* Status date/time.
* Recipient name (for some status types).

Other information might be required from a business perspective according to the contractual relationship with a supplier.

### Basic Information

The `TransmissionID` element should contain a supplier-defined unique identifier *for the transmission*, not for the status. Ideally, if you transmitted the same information multiple times, each transmission would contain a different `TransmissionID`. This element will be returned to you in the XML response. (This is largely unnecessary for synchronous HTTP responses, but if we ever support responses using FTP, this ID will be critical to match asynchronous the response to the original transmission.)

The `TransmissionGeneratedUTC` is the date at which the transmission was formed, in the UTC time zone. `TransmissionGenerated` is the date at which the transmission was formed, in your time zone. These dates should be represented in ISO 8601 format. The time zone is optional but encouraged. We use these to help understand when the transmission was generated for troubleshooting purposes. These dates should *not* be a copy of the date on which the status occurred. (If the `TransmissionGenerated` element contained a time zone reference, we would not need a separate UTC version, but in case time zone can't be added, we find having both useful for developer troubleshooting.)

### Authentication

The `Authentication` element contains a `UserName` and `PasswordHash`.

The authorization system is based on our [Clarity](http://MFSClarity.com/) supplier web application. In order to submit a status for a manifest, the Clarity user specified in the transmission must have access to the Agents application for the manifest's controlling supplier. Typically, the supplier developer should get credentials from the supplier's business contacts, but they can also be provided by us. We recommend that a business contact create a separate user in Clarity for the EDI process. Clarity usernames must be emails, and we recommend that the email used be for an IT group who will be responsible for maintaining the integration.

The authentication information is passed as part of the content of the message rather than as part of the connection process. There is no authentication involved in the connection to the HTTP endpoint itself (e.g. BASIC HTTP authentication), and the authentication involved in an FTP connection is in addition to what is contained within the file contents.

In order to avoid sending the username and password together in the transmissions, we instead designed a dynamic hashing concept which obscures the password and makes it difficult to use the same hash in a subsequent (e.g. spoofed) transmission. The `PasswordHash` should be the SHA-1 hash of the following values:

  > `UserName` + `|` + `TransmissionID` + `|` + `TransmissionGeneratedUTC` + `|` + `Password`

Thus plus signs represent concatenation. The values must be exactly those used in the transmission, with no reformatting. If there is a leading 0 in the `TransmissionID`, it must be preserved. The exact text format of the data in `TransmissionGeneratedUTC` must be used, not a semantically equivalent value. The `Password` should be the password for the supplier Clarity user specified in `UserName`.

### StatusSubmissions

The `StatusSubmissions` element contains one or more `StatusSubmission` elements.

Each of those elements contains a `SubmissionID`. Ideally this would be unique across all submissions, or at least the combination of `TransmissionID` and `SubmissionID` would be unique across all transmissions. It is used for developer troubleshooting. It is returned in our Response in the `SubmissionID` element.

#### Shipment

We regret the choice of the name "Shipment" here. "Manifest" is a better term. Customers request us to move shipments. We consolidate shipments together into manifests and contract with suppliers to move those groups of shipments. Our [outbound Manifest transmission system](https://github.com/MFSTech/GetManifests) properly distinguishes between the Manifest, which represents the official interaction with the supplier, and the Shipments, which are provided merely for additional information. If it weren't for legacy support, we would change this "Shipment" element name to "Manifest" to eliminate confusion. Indeed, within the `ControlNumbers` section, we refer to the `ManifestID` and the `ManifestTranID`.

Contained elements:

* `KeyDates`
* `ControlNumbers`
* `Pieces`
* `Charges`
* `ShipmentSummary`

Much of the information in this section is optional.

##### KeyDates

This section is not required from a technical perspective.

* `ETD` - The Estimated Time of Departure. For Manifests of type MAWB-- or any manifest representing long distance travel rather than a local pickup or delivery-- this should represent the date and time the freight will be leaving. ISO 8601 format in the local time is preferred. For other manifests, this information is merely logged.
* `ETA` - The Estimated Time of Arrival. For Manifests of type MAWB-- or any manifest representing long distance travel rather than a local pickup or delivery-- this should represent the date and time the freight will be arriving at destination. ISO 8601 format in the local time is preferred. For other manifests, this information is merely logged.

##### ControlNumbers

A submission will fail if we cannot find a manifest in our system to which it corresponds, so use of this section is required.

* `VendorOrderID` - This is a deprecated element and should not be used in the future. Use `ManifestID` instead.
* `ManifestID` - This is the primary textual identifier for a manifest in our system. We generally set it to the primary control number for the manifest as defined by the supplier. (For MAWBs, this is usually the carrier MAWB number. For agent pickups and deliveries, this is often the PRO number.) If you are using our [outbound Manifest transmission system](https://github.com/MFSTech/GetManifests), and everything is configured correctly, then this `ManifestID` will be equal to the `VendorManifestControlNumber` you are sending back in your response. If you are not sending that element back, then this `ManifestID` will be the same as the `ManifestID` which our system generated and we are sending you in that outbound transmission.
* `ManifestTranID` - This is the internal, numeric identity value for a manifest in our system. It is unique and inviolate. It will match the element of the same name in the [outbound Manifest transmission](https://github.com/MFSTech/GetManifests). This is the best method to specify a particular manifest. However, while our operations staff will understand what this Transaction ID is, the operations staff at our supplier might not, and it is not a frequently used number by humans.
* `ShipmentID` - This was designed to receive the primary textual identifier for a *shipment* (not manifest) in our system. Some of our suppliers, especially pickup and delivery agents, do not track our `ManifestID` or our `ManifestTranID`. Instead, they conflate the concept of a manifest and a shipment and just use the primary shipment number (otherwise known as our BOL or tracking number) in human and system communication. This is a pragmatic approach for those of our suppliers who predominantly work with deliveries, where there is usually a single shipment for each manifest, but it falls apart for more complex manifests.
* `ShipmentTranID` - This was to contain the internal, numeric, inviolate identity value for a *shipment* in our system. Using this has the same limitations as the `ShipmentID`.

Because of the evolution of this system over time, and because of the loose distinctions made between shipments and manifests by our and our suppliers' operations staffs, we have taken a liberal approach to matching a manifest in our system to a status submission based on these control numbers. We strongly recommend the use of the `ManifestTranID` or `ManifestID`, but the system might find a match using the other numbers, which might be the only option for some suppliers who are not using our [outbound Manifest transmission system](https://github.com/MFSTech/GetManifests).

##### Pieces

This section is optional. Providing this information allows us to be alerted when there might be a discrepancy between the supplier's and our records with respect to the configuration of pieces on a manifest.

This element should contain a collection of one or more `Piece` elements, each of which has the following sub-elements. Each `Piece` element need not represent an individual physical shipping piece. Identical pieces can be grouped together with the count placed in the `PieceCount` element.

* `PieceID` - This was intended to be the unique identifier of a piece in our system, which value would have been fed from [outbound Manifest transmission system](https://github.com/MFSTech/GetManifests). This would allow us to understand which particular pieces had differences between our systems. However, it is not currently used.
* `PieceCount` - This represents the number of identical pieces on the manifest which fit the rest of the characteristics in this `Piece` element. This is *not* to be used as a `Said to Contain` or `Carton Count` value which would represent a number of boxes which have been combined into a single shipping piece (such as small boxes banded to a large pallet shipping unit).
* `PieceLength` - The length, in inches, of the piece.
* `PieceWidth` - The width, in inches, of the piece.
* `PieceHeight` - The height, in inches, of the piece.
* `PieceWeightActual` - The actual weight, in pounds, of the piece. If there are multiple, identical pieces combined and represented using the `PieceCount` element, then this should be the weight of *each* of the individual identical pieces, not the total.
* `PieceWeightDimensional` - This should be the calculated dimensional weight, in pound units, of the piece. As with `PieceWeightActual`, this should be per individual piece. Dimensional weight is a weight-based representation of the volume of a piece. It is calculated by taking the product of the length, width, and height and dividing by a negotiated factor. Typically, for ground shipments, the factor is 250. The dimensional weight is not relevant to all contracts; some freight charges are calculated by actual weight only.

While all of this information is logged and valuable for researching discrepancies, the existence of a discrepancy is detected based on the totals of the values here and in the `ShipmentSummary` section.

##### Charges

This section is optional.  Providing this information allows us to be alerted when there might be a discrepancy between the supplier's and our records with respect to the impending costs related to the manifest.

This element should contain a collection of one or more `Charge` elements, each of which has the following sub-elements.

* `VendorChargeType` - This was designed to contain the supplier's code for this type of charge. We have the ability to map these codes to charges we might expect. We specifically look for freight and fuel charges as those are the most standardized across vendors.
* `ChargeType` - This would be our code for this charge type. It would be the same as what we sent in `CostCode` field in the [outbound Manifest transmission system](https://github.com/MFSTech/GetManifests).
* `ChargeDesc` - This should be used for a human-friendly name for or description of the charge.
* `ChargeRate` - This is the Rate of the charge which, when multiplied by the Value, results in the Amount. This is typically a negotiated, contractual amount based on various manifest characteristics such as geography.
* `ChargeValue` - This is the Value of the charge which, when multiplied by the Rate, results in the Amount. This is typically derived from a characteristic of the manifest, such as its total weight or its freight charge.
* `ChargeMin` - This is the minimum amount for this type of charge.
* `ChargeAmount` - This is the actual dollar value (in USD) for this charge. It should generally be the product of the Rate and Value (perhaps scaled by a factor, e.g. divided by 100). The Rate, Value, and Minimum are informational, but we base logic on this Amount.

While all of this information is logged and valuable for researching discrepancies, the existence of a discrepancy is detected based on the totals of the values here and in the `ShipmentSummary` section.

##### ShipmentSummary

This section is optional. For those who are unable or unwilling to specify the specific piece and charge information, it serves as an easy way to signal total pieces, weight, and costs, which can also be used to alert our operations staff.

* `TotalPieces` - The total number of all pieces on the manifest (should equal the sum of all the `PieceCount` elements).
* `TotalWeightActual` - The total actual weight (in pounds) of all pieces on the manifest. It should equal the sum of all of the mathematical products of the `PieceCount` and `PieceWeightActual` elements.
* `TotalWeightDim` - The total dimensional weight (in pounds) of all pieces on the manifest. It should equal the sum of all of the mathematical products of the `PieceCount` and `PieceWeightDimensional` elements.
* `TotalWeightChargeable` - This should be the greater of the `TotalWeightActual` and `TotalWeightDim` amounts. There is no concept of chargeable weight at the piece level, as by convention it is calculated only at the aggregate level. It is *not* the sum of the greater of the actual or dimensional weights for each piece, but rather the greater of the sums.
* `TotalCharges` - This should be the total charges for the entire manifest. It should equal the sum of the `ChargeAmount` elements.

#### Status

From a technical perspective, `StatusDateTime` must be specified, as must either `VendorStatusType` or `StatusType`. For some status types, `StatusNotes` must also be specified.

* `VendorStatusID` - This should be a value which uniquely identifies this status in your system. We use this to try to detect updates. It is also valuable for troubleshooting.
* `StatusID` - This can be used to specify the particular status record in our system to indicate that is an update. This is the value we return in the `SubmissionStatusID` in our response.
* `LocationCode` - This should be used to specify the location where the status event occurred. For MAWB manifests, this should be IATA code for the nearest major airport to the terminal facility where the status occurred. For other manifest types, this is free form text and primarily used by our operations staff when investigating exceptions. We anticipate using this or another set of elements for GPS coordinates in the future.
* `VendorStatusType` - This should be your code to identify the type of status event which occurred. We will use this code to map to one of our official status types.
* `VendorStatusDesc` - This should be the human-friendly description of the status event. It is valuable to send this even if you specify a status code, as sometimes statuses cannot be mapped, but a human-friendly log can still be used by our operations staff to understand what is occurring with a manifest.
* `StatusType` - If you know our status code, you can specify it here instead of using your code in `VendorStatusType` and having us map it. However, our mapping might be more flexible than what your system could accomplish, and using our mapping allows us to control when and how we apply statuses. We generally recommend that you use your code in `VendorStatusType` and allow us to perform the mapping. Our codes are listed in the `StatusTypeID` element in our [status code list](http://edi.mfsclarity.com/EDI/XMLIn.aspx?Process=GetStatusTypes) which is referenced on our [Codes page](https://github.com/MFSTech/EDIAPI/blob/master/Codes.md).
* `StatusDateTime` - This is the *local* date and time the status event occurred, not (merely) when it was recorded. It should be represented in ISO 8601 format using the local time zone, ideally including the offset. This is a fundamental and required element.
* `StatusEntryDateTime` - This is the date and time the status event was recorded. Ideally, it would be in the same time zone as `StatusDateTime` (unless each has an offset, in which case it doesn't matter as much) as it is sometimes used to determine entry timeliness.
* `StatusException` - This text field can be used to note exception information which might be valuable to our operations staff. It will be highlighted in various parts of our application.
* `StatusNotes` - This text field is traditionally used to store the name of the person receiving the goods during the event the status entry represents. The system requires a value (and restricts some values) for some status types. From a business rule perspective, a delivery supplier should provide the name of the recipient for a delivery status such as a POD (Proof of Delivery).

Transmission
------------

The XML content can be transmitted via HTTP or FTP. HTTP transmission is processed synchronously and a response will be provided. FTP transmissions are processed asynchronously and no response will be provided. HTTP is the preferred transmission mechanism.

### HTTP

The production endpoint for receiving the XML document is:

[http://EDI.MFSClarity.com/EDI/XMLIn.aspx?Process=VendorManifestStatus](http://EDI.MFSClarity.com/EDI/XMLIn.aspx?Process=VendorManifestStatus)

An SSL enabled endpoint is not available at this time.

That ASP.NET endpoint handles multiple kinds of transmissions which are identified with the URL parameter Process. `VendorManifestStatus` identifies the XML document as a freight status from a supplier.

The XML document must be sent "raw" as the body of the HTTP POST. It should not be sent as a parameter in the URL. It should also not be sent as a parameter in the POST body using the traditional application/x-www-form-urlencoded content type. While the content-type header is not required to be specified as `text/xml` or `application/xml`, the content should be formatted as such.

#### Feeder

To separate transmission concerns from content evaluation, we have created a "feeder" application for use by developers. Navigate to:

[http://EDI.MFSClarity.com/EDI/XMLInFeeder.aspx](http://EDI.MFSClarity.com/EDI/XMLInFeeder.aspx)

And then paste the candidate XML into the textbox. Be sure to specify the Process parameter by putting `VendorManifestStatus` in the Process text field. This page will feed the XML into the above XMLIn endpoint in the appropriate manner and will expose the response in human-readable form.

(While a developer could configure an application to use this Feeder tool automatically, by using traditional a HTTP POST with form-encoded parameters, to avoid having to directly POST XML to the real endpoint, we officially discourage and will not support that automation. We reserve the right to adjust this developer-facing tool in ways which could break any such use.)

#### HTTP Status Codes

Unfortunately, we are not making good use of HTTP response status codes. Generally, the status code will be 500 (Internal Server Error) if anything is amiss, though it doesn't necessarily mean that there is an error with our system. For example, failing to specify valid XML will result in a 500 status, when something more like a 400 would be more appropriate.

More frustratingly, a 200 status does not necessarily mean anything useful happened. For example, passing `<a>Hello!</a>` will meet with a 200 status code response with no errors indicated, even though nothing valuable occurred. Furthermore, you will still receive a 200 status code even if there are problems processing your submissions. For example, if you supply invalid credentials, the response content will explain that and indicate a problem, but the overall HTTP status code will still be 200.

Importantly, an XML-formatted response is almost always returned, even when the HTTP status is 500. Only in cases of significant failure with our systems would an XML response not be provided. It is important to mine the XML response for information, regardless of the HTTP status code, as the text within it provides good troubleshooting information.

Our long term goal is to improve HTTP status code generation so that they provide better information which automation can rely upon.

#### Empty Transmissions

If you simply navigate to the endpoint URL using a browser, you will be making an HTTP GET request. You will not be providing any XML in an HTTP POST, and the system will indicate an error by an HTTP status 500 (see above) and provide an XML response containing the text "System error processing transmission: XML is empty.". This does not mean our system is malfunctioning. If you want to manually test the system, you should send some valid XML via the XMLFeeder system.

#### Development

We have a non-production environment for suppliers and customer developers to use to develop and test their integration processes. The development endpoint is:

[http://Demo.MFSClarity.com/EDI/XMLIn.aspx](http://Demo.MFSClarity.com/EDI/XMLIn.aspx)

Similarly, the development "feeder" endpoint is:

[http://demo.mfsclarity.com/EDI/XMLInFeeder.aspx](http://demo.mfsclarity.com/EDI/XMLInFeeder.aspx)

There are a couple of challenges using this environment. First, it isn't used often, and it isn't supplied production-grade resources. Therefore, it tends to be quite slow, especially when used for the first time when code and data is not compiled and cached. Therefore, if you get an error like this from the feeder:

> ERROR - [cmdGO_Click].
> 
> The operation timed out 

Just keep trying. Or try reducing the number of `StatusSubmission` elements or other non-required elements.

Secondly, the data is only refreshed from production intermittently every few months. Therefore, if you are developing for a new supplier of ours, that new supplier and related credentials are likely not in the development environment. Furthermore, even if the supplier is configured, sending statuses successfully requires existing manifests assigned to that vendor. Therefore, there is some setup required by our IT or operations staff.

Often, after confirming syntax and formatting issues, we will encourage a transition to production. Until we set up mapping in our system, the transmissions go into an informational log for our staff to review manually for additional background, so even mistakes are relatively harmless.

#### Response

When issuing an HTTP request to our endpoint, the HTTP response will almost always contain XML. The response content-type is currently set to "text/xml". The Feeder tool displays this response XML-- in raw format and HTML-encoded format-- and also exposes the HTTP status. 

The root of the response will be `StatusEntryResponse` and it will contain an `ErrorID` and `ErrorText`. The presence of data in the error elements indicate that something is went wrong with the submission-- either with the content or our processing of it. The absence of data in those top-level error elements does not mean that the transmission was successful.

The responses will also contain a `VendorTransmissionID`, which echoes the `TransmissionID` you send in your transmission, and an `MFSTransmissionID`, which is our unique identifier for your transmission.

Unfortunately, there is some inconsistency in our responses. For example, when you send no XML, the timestamp element names are `ResponseGeneratedUTC` and `ResponseGeneratedLocal`, but when you send invalid XML, the element names are `ResponseTimeUTC` and `ResponseTimeLocal`. Our long term plan is to resolve this inconsistency.

When a well-formed transmission is sent, it will include one or more status submission. The results of those status submissions are presented in the XML response. There will be one or more associated `Submission` elements in a parent `Submissions` element, each of which will have `SubmissionErrorID` and `SubmissionErrorText` elements. Those error elements being empty is a good indication that a particular status submission (within the overall transmission) was successful.

Proof that a status submission was successfully mapped to a status in our system is a non-zero `SubmissionStatusID`. However, we might not map all vendor status types to actual statuses in our system, but rather merely keep them in a log. In that case, there would be no `SubmissionStatusID`, but it's still effectively a success.

### FTP

We can also accept these transmissions as XML within files. Each file must contain only one transmission (just as each HTTP request must contain only one transmission), though transmissions may contain multiple status submissions for multiple manifests.

We can host an FTP site into which you would upload files or we can connect to an FTP site you host from which we would download and delete files. The FTP site can be the same or different than that used for [Manifest transmissions](https://github.com/MFSTech/GetManifests). Also, choosing FTP for one interaction with us (e.g. manifest transmissions) does not require using FTP for other interactions. If desired, you can send some transmissions via files and other transmissions via HTTP.

SFTP is also supported.

However, responses are *not* supported using FTP at this time. Therefore, your submissions are done blindly. You will not know if they succeeded except through other channels. If we eventually support responses using FTP, it will require correlating data between your transmission and our response. We intend that correlating data to be your supplied `TransmissionID`, which we return as `VendorTransmissionID`. That requires it to be unique among your transmissions.

Miscellany
----------

* For the purposes of this document, "you" are the technical representative for the freight movement supplier (e.g. trucking company) and "we" are Pilot Freight Services, who is contracting with the supplier for them to perform services for us.
* A single transmission can contain multiple status submissions, but multiple statuses for the same manifest cannot be combined in the same status submission. Multiple statuses for the same manifest must be separated into multiple status submissions, with the manifest-level information repeated in each status submission.
* Many of our customers ask us to perform "advance exchange" or "whole unit exchange" services, which involve us delivering a replacement product to an end user and recovering an existing product from them at the same time. We then return that existing product to our customer. We break this service into two shipments-- one from our customer's warehouse to the end user and another from that end user back to the customer's warehouse. Because we treat those shipments separately, what might be a single "job" from our supplier's perspective (dropping off the new and bring back the original) are actually two manifests from our perspective. Therefore, the supplier would have to provide both a Proof of Delivery status on the outbound (Delivery) manifest and a Proof of Pickup status on the return (Pickup) manifest.
