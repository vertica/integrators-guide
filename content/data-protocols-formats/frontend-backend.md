---
title: "Frontend and backend protocols"
linkTitle: "Frontend and backend protocols"
weight: 20
---

Vertica uses a message-based protocol for communication between frontends and backends (clients and servers). The protocol is supported over TCP/IP sockets. This document describes version 3.15 of the protocol.

For purposes of the protocol, the terms "backend" and "server" are interchangeable; likewise "frontend" and "client" are interchangeable. See [vertica-python](https://github.com/vertica/vertica-python) for a frontend implementation reference of the protocol.



## Overview

The protocol has separate phases for startup and normal operation.

In the startup phase, the frontend opens a connection to the backend and authenticates itself to the satisfaction of the backend. (This might involve a single message, or multiple messages depending on the authentication method being used.) If all goes well, the backend then sends status information to the frontend, and finally enters normal operation. Except for the initial startup-request message, this part of the protocol is driven by the backend.

During normal operation, the frontend sends queries and other commands to the backend, and the backend sends back query results and other responses. There are a few cases wherein the backend will send unsolicited messages, but for the most part this portion of a session is driven by frontend requests.

Termination of the session is normally by frontend choice, but can be forced by the backend in certain cases. In any case, when the backend closes the connection, it will roll back any open (incomplete) transaction before exiting.

Within normal operation, SQL commands can be executed through either of two sub-protocols. In the "simple query" protocol, the frontend just sends a textual query string, which is parsed and immediately executed by the backend. In the "extended query" protocol, processing of queries is separated into multiple steps: parsing, binding of parameter values, and execution. This offers flexibility and performance benefits, at the cost of extra complexity.

Normal operation has additional sub-protocols for special operations such as COPY.

All communication is through a stream of messages. The byte order of all messages are big-endian (A special case in [WriteFile](#writefile-o) message). All data sending to or coming from the backend are represented as UTF-8.

### Formats and Format Codes

Data of a particular SQL data type might be transmitted in either "text" format or "binary" format. The desired format for any value is specified by a *format code*. Text has format code zero, binary has format code one, and all other format codes are reserved for future definition.

The text representation of values is whatever strings are produced and accepted by the input/output conversion functions for the particular data type. For example, the value of DATE/TIMESTAMP data is a date/timestamp in proleptic Gregorian calendar.

#### Binary Format Encoding For Each Data Type

<table>
   <thead>
      <tr>
         <th>
            <p>Data Type</p>
         </th>
         <th>
            <p>Length (bytes)</p>
         </th>
         <th>
            <p>Data Content</p>
         </th>
      </tr>
   </thead>
   <tbody>
      <tr>
         <td>
            <p>BOOLEAN</p>
         </td>
         <td>
            <p>1</p>
         </td>
         <td>
            <p>0 for false, 1 for true.</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>CHAR</p>
         </td>
         <td>
            <p>User-specified</p>
         </td>
         <td>
            <p>A sequence of UTF-8 encoding bytes.</p>
            <ul>
               <li> UTF-8 strings can contain multi-byte characters. Therefore, number
               of characters in the string may not equal the number of bytes.
               </li>
               <li> Strings shorter than the specified length are right-padded with spaces.
               </li>
            </ul>
         </td>
      </tr>
      <tr>
         <td>
            <p>VARCHAR</p>
         </td>
         <td rowspan="2">
            <p>Variable-length</p>
         </td>
         <td rowspan="2">
            <p>A sequence of UTF-8 encoding bytes.<br>Remember that UTF-8 strings
               can contain multi-byte characters. Therefore, number of characters in
               the string may not equal the number of bytes.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>LONG VARCHAR</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>INTEGER</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer.</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>FLOAT</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>Encoded in IEEE-754 format.</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>NUMERIC</p>
         </td>
         <td>
            <p><i>N</i> <small>(N = (precision//19+1)*8)</small></p>
         </td>
         <td>
            <p><i>N</i>-byte signed integer containing the unscaled value. Scale is
               calculated separately with <a href="#rowdescription-t">RowDescription</a>'s type modifier.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>DATE</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer containing the <a
               href="https://en.wikipedia.org/wiki/Julian_day">Julian day number</a>
               (JDN) for the date.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>INTERVAL</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer containing the number of microseconds in the
               interval.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>INTERVALYM</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer containing the number of months in the
               interval.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>TIME</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer containing the number of microseconds since
               midnight in the UTC time zone.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>TIMETZ</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit value where:</p>
            <ul>
               <li>Upper 40 bits contain the number of microseconds since midnight in
                  the UTC time zone.
               </li>
               <li>Lower 24 bits contain time zone as the UTC offset in seconds
                  calculated as follows:<br><i>Actual time zone</i> = 86400<small>(24hrs)</small> - this 3-byte value as integer
               </li>
            </ul>
         </td>
      </tr>
      <tr>
         <td>
            <p>TIMESTAMP</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer containing the number of microseconds since Julian
               day: <i>Jan 01 2000 00:00:00</i>.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>TIMESTAMPTZ</p>
         </td>
         <td>
            <p>8</p>
         </td>
         <td>
            <p>64-bit integer containing the number of microseconds since Julian
               day: <i>Jan 01 2000 00:00:00</i> in the UTC timezone.
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>UUID</p>
         </td>
         <td>
            <p>16</p>
         </td>
         <td>
            <p>A 128-bit value interpreted as UUID.</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>BINARY</p>
         </td>
         <td>
            <p>User-specified</p>
         </td>
         <td>
            <p>A sequence of bytes. If the value is smaller than the specified
               length, the remainder should be filled with nulls (\x00).
            </p>
         </td>
      </tr>
      <tr>
         <td>
            <p>VARBINARY</p>
         </td>
         <td rowspan="2">
            <p>Variable-length</p>
         </td>
         <td rowspan="2">
            <p>A sequence of bytes.</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>LONG VARBINARY</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>Complex types (ARRAY, MAP, ROW, SET)</p>
         </td>
         <td>
            <p>/</p>
         </td>
         <td>
            <p>Data content is the same as TEXT format.</p>
         </td>
      </tr>
   </tbody>
</table>

> **Note**: While PostgreSQL lets you change the format on a per-column/type basis in the protocol, Vertica currently does not honor that and instead does it as a session-level setting. So unfortunately that means its an all-or-nothing implementation - all data types must be implemented. To opt into binary format protocol for data reads from backend (queries), the frontend have to set the [StartupRequest](#startuprequest) message to include parameter 'binary_data_protocol' set to '1' ('0' is the default, which is the text format protocol), and set the [Bind](#bind-b) message's last two fields (result-column format code) as '1'. No Vertica clients currently support data writes (server-prepared statement bind values) to the backend in binary format.

## Message Flow

This section describes the message flow and the semantics of each message type. (Details of the exact representation of each message appear in Section [Message Formats](#message-formats).)

### Start-up

To begin a session, a frontend opens a connection over TCP/IP to the server. With a TCP socket, the frontend does the following steps:

#### Connection load balancing
  
A backend with load balancing enabled can redirect connections based on the connection's origin. To initiate a connection load balancing, the frontend sends a [LoadBalanceRequest](#loadbalancerequest) message. The server then responds with a [LoadBalanceResponse](#loadbalanceresponse-yn) message, the first byte of the message containing ***Y*** or ***N***, indicating that it is willing or unwilling to perform load balancing, respectively. To continue after ***Y***, the frontend must create a new TCP socket, which connects to the new host/port specified in the [LoadBalanceResponse 'Y'](#loadbalanceresponse-yn) message. To continue  ***N***, send the usual [SSLRequest](#sslrequest) message and [Startup](#startuprequest) message and proceed without connection load balancing.
  
To enable connection load balancing on the server side, please see Vertica documentation.

*Workload routing* is different from connection load balancing, which happens after [authentication](#startup-message-and-authentication). The node the client connects to would forward traffic to the execution node. In other words, the connection node acts as a proxy between the client and the execution node.

#### SSL session encryption

Frontend/backend communications can be encrypted using TLS/SSL. This provides communication security in environments where attackers might be able to capture the session traffic.
  
To initiate an SSL-encrypted connection, the frontend sends an [SSLRequest](#sslrequest) message. The server then responds with a single byte containing ***S*** or ***N***, indicating that it is willing or unwilling to perform SSL, respectively. The frontend might close the connection at this point if it is dissatisfied with the response. To continue after ***S***, perform an SSL startup handshake (not described here, part of the SSL specification) with the server. If this is successful, all subsequent data will be SSL-encrypted. To continue after ***N***, send the usual [Startup](#startuprequest) message and proceed without encryption.

To enable TLS/SSL communications on the server side, please see Vertica documentation.

#### Startup message and authentication
  
Once the socket setup is done, the frontend sends a [StartupRequest](#startuprequest) message to the backend. This message includes the names of the user and of the database the user wants to connect to; it also identifies the particular protocol version to be used. Optionally, the [StartupRequest](#startuprequest) message can include additional settings for run-time parameters. The server then uses this information to determine whether the connection is provisionally acceptable, and what additional authentication is required (if any).
  
The server then sends an appropriate authentication request message, to which the frontend must reply with an appropriate authentication response message (such as a [Password](#password-p)).

The authentication cycle ends with the server either rejecting the connection attempt ([ErrorResponse](#errorresponse-e)), or sending [AuthenticationOk](#authenticationok-r).

The possible messages from the server in this phase are:

[ErrorResponse](#errorresponse-e)
: The connection attempt has been rejected. The server then immediately closes the connection.

[AuthenticationOk](#authenticationok-r)
: The authentication exchange is successfully completed.

[AuthenticationCleartextPassword](#authenticationcleartextpassword-r)
: The frontend must now send a [Password](#password-p) message containing the password in clear-text form. If this is the correct password, the server responds with an [AuthenticationOk](#authenticationok-r), otherwise it responds with an [ErrorResponse](#errorresponse-e). 

[AuthenticationGSS](#authenticationgss-r)
: The frontend must now initiate a GSSAPI negotiation. The frontend will send a [Password](#password-p) message with the first part of the GSSAPI data stream in response to this. If further messages are needed, the server will respond with [AuthenticationGSSContinue](#authenticationgsscontinue-r). See [GSS-API/Kerberos Authentication](#gss-api--kerberos-authentication).

[AuthenticationGSSContinue](#authenticationgsscontinue-r)
: This message contains the response data from the previous step of GSSAPI negotiation (AuthenticationGSS, or a previous AuthenticationGSSContinue). If the GSSAPI data in this message indicates more data is needed to complete the authentication, the frontend must send that data as another [Password](#password-p) message. If GSSAPI authentication is completed by this message, the server will next send [AuthenticationOk](#authenticationok-r) to indicate successful authentication or [ErrorResponse](#errorresponse-e) to indicate failure.

[AuthenticationPasswordExpired](#authenticationpasswordexpired-r)
: The connection attempt has been rejected as the password is expired.

[AuthenticationPasswordGrace](#authenticationpasswordgrace-r)
: This message informs the frontend that the password will expire soon. The frontend can display a warning message. Then, the frontend must wait for further messages from the server.

[AuthenticationMD5Password](#authenticationmd5password-r) or [AuthenticationHashMD5Password](#authenticationhashmd5password-r)
: The frontend must now send a [Password](#password-p) message containing the password (with username) encrypted via MD5, then encrypted again using the 4-byte random salt specified in this message. If this is the correct password, the server responds with an [AuthenticationOk](#authenticationok-r), otherwise it responds with an [ErrorResponse](#errorresponse-e).

[AuthenticationHashPassword](#authenticationhashpassword-r) or [AuthenticationHashSHA512Password](#authenticationhashsha512password-r)
: The frontend must now send a [Password](#password-p) message containing the password (with the 16-byte user salt specified in this message) encrypted via SHA512, then encrypted again using the 4-byte random salt specified in this message. If this is the correct password, the server responds with an [AuthenticationOk](#authenticationok-r), otherwise it responds with an [ErrorResponse](#errorresponse-e).
 
If the frontend does not support the authentication method requested by the server, then it should immediately close the connection.

After having received AuthenticationOk, the frontend must wait for further messages from the server. In this phase a backend process is being started, and the frontend is just an interested bystander. It is still possible for the start-up attempt to fail (ErrorResponse), but in the normal case the backend will send some [ParameterStatus](#parameterstatus-s) messages, [BackendKeyData](#backendkeydata-k), and finally [ReadyForQuery](#readyforquery-z).

During this phase the backend will attempt to apply any additional run-time parameter settings that were given in the [StartupRequest](#startuprequest) message. If successful, these values become session defaults. An error causes [ErrorResponse](#errorresponse-e) and exit.

The possible messages from the backend in this phase are:

[ParameterStatus](#parameterstatus-s)
: This message informs the frontend about the current (initial) setting of backend parameters, such as client_locale or auto_commit. The frontend can ignore this message, or record the settings for its future use. The frontend should not respond to this message, but should continue listening for a ReadyForQuery message.

[BackendKeyData](#backendkeydata-k)
: This message provides secret-key data that the frontend must save if it wants to be able to [issue cancel requests](#canceling-requests-in-progress) later. The frontend should not respond to this message, but should continue listening for a ReadyForQuery message.

[ReadyForQuery](#readyforquery-z)
: Start-up is completed. The frontend can now issue commands.

[ErrorResponse](#errorresponse-e)
: Start-up failed. The connection is closed after sending this message.

[NoticeResponse](#noticeresponse-n)
: A warning message has been issued. The frontend should display the message but continue listening for ReadyForQuery or ErrorResponse.

The [ReadyForQuery](#readyforquery-z) message is the same one that the backend will issue after each command cycle. Depending on the coding needs of the frontend, it is reasonable to consider [ReadyForQuery](#readyforquery-z) as starting a command cycle, or to consider [ReadyForQuery](#readyforquery-z) as ending the start-up phase and each subsequent command cycle.

### Authentication

#### GSS-API / Kerberos Authentication

GSS-API is _Generic Security Service API (RFC 2744)_. It provides a common interface for accessing different security services. One of the most popular security services available for GSS-API is the Kerberos v5.

Here are the most basic steps taken to authenticate in a Kerberized environment:

1. Client requests an authentication ticket (TGT) from the Key Distribution Center (KDC).
2. The KDC verifies the credentials and sends back an encrypted TGT and session key.
3. The TGT is encrypted using the Ticket Granting Service (TGS) secret key.
4. The client stores the TGT and when it expires the local session manager will request another TGT (this process is transparent to the user).


`kinit` in Linux is a command often used for obtaining or caching/renewing a Kerberos ticket-granting ticket (TGT).


When the client requests access to a server, this is the process:

1. The client sends a [StartupRequest](#startuprequest) message to the server.
2. The server sends back a [AuthenticationGSS](#authenticationgss-r) message to request GSS-API negotiation.
3. The client sends the current TGT to the TGS with the Service Principal Name (SPN) of the Vertica server.
4. The KDC verifies the TGT of the user and that the user has access to the Vertica server service.
5. TGS sends a valid session key for the service to the client.
6. Client forwards the session key (within a [Password](#password-p) message) to the server to prove the user has access.
7. If further messages are needed, the server will respond with a [AuthenticationGSSContinue](#authenticationgsscontinue-r) message. The client sends data as another [Password](#password-p) message.
8. The server grants access and sends a [AuthenticationOk](#authenticationok-r); or sends a [ErrorResponse](#errorresponse-e) to indicate failure.


![Example message flow of GSS authentication](/images/data-protocols-formats/frontent-backend/GSSAuth_Msg_flow.png "Example message flow of GSS authentication")

#### OAuth2 authentication

OAuth lets client applications verify themselves to and receive an OAuth access token from an identity provider (IDP) like Keycloak or Okta. These client applications can then pass the access token to authenticate to Vertica server as a substitute for username & password.

A client user should first configure the IDP and get the following parameters:

- ***client_id***: ID of the confidential client (application) registered in IDP. This is used by server to call introspection API to retrieve user grants; and used by client to retrieve tokens.

- ***client_secret***: The secret of the confidential client (application) registered in IDP. This is used by server to call introspection API to retrieve user grants; and used by client to retrieve tokens.

- ***introspect_url***: API used by server to introspect access token. This URL is exposed by IDP and must be accessible from server.

- ***token_url***: API used by client to refresh the token. This URL is exposed by IDP and must be accessible from client.


Then in the server, create an OAuth authentication record with above parameters, and grant it to the user.

The client application or user call the IDP token API to retrieve the following parameters:

- ***access_token***: Access tokens are the thing that applications use to make connection requests on behalf of a user.

- ***refresh_token***: Refresh tokens are used to obtain a new access token.

<figure>
  <img src="/images/data-protocols-formats/frontent-backend/OAuth_Msg_flow.png"                                   
       title="Example message flow of OAuth authentication (Protocol 3.11)"
       width="400"
       alt="Example message flow of OAuth authentication" />
  <figcaption aria-hidden="true">Example message flow of OAuth authentication (Protocol 3.11)</figcaption>
</figure>

<figure>
  <img src="/images/data-protocols-formats/frontent-backend/OAuth_refresh_Msg_flow.png"                                   
       title="Example message flow of OAuth access token refresh (Protocol 3.11)"
       width="400"
       alt="Example message flow of OAuth access token refresh" />
  <figcaption aria-hidden="true">Example message flow of OAuth access token refresh (Protocol 3.11)</figcaption>
</figure>


#### (Protocol 3.11)

In the [StartupRequest](#startuprequest), only access_token is required to be specified in **oauth_access_token** parameter, no client id/secret is included in the request. The 'user' parameter can be an empty string. The backend validates token using OAuth Introspect query. In case oauth_access_token is valid and permissions are sufficient, the backend sends [AuthenticationOk](#authenticationok-r), otherwise it responds with an [ErrorResponse](#errorresponse-e). Token refresh flow can be triggered by frontend after the token validation (if token introspection fails).

> **Note**: It is not recommended for clients to implement this protocol version by default. [Protocol 3.12](#protocol-312) is the right way to pass oauth_access_token to servers. For backwards compatibility with v11.1SP1 - v12.0SP1 servers, clients can implement a parameter to include the oauth_access_token in the startup request.

#### (Protocol 3.12)

In the [StartupRequest](#startuprequest), if the 'auth_category' parameter is specified as "OAuth", the server will send the client an [AuthenticationOAuth](#authenticationoauth-r) message. The client will respond with a [Password](#password-p) message containing an OAuth access token. The backend validates token using OAuth Introspect query. In case the access token is valid and permissions are sufficient, the backend sends [AuthenticationOk](#authenticationok-r), otherwise it responds with an [ErrorResponse](#errorresponse-e). Token refresh flow can be triggered by frontend after the token validation (if token introspection fails).

If 'auth_category' parameter is not set, the server still check for the token in the 'oauth_access_token' parameter of [StartupRequest](#startuprequest) message.

<figure>
  <img src="/images/data-protocols-formats/frontent-backend/OAuth_Msg_flow_312.png"                                   
       title="Example message flow of OAuth authentication (Protocol 3.12)"
       alt="Example message flow of OAuth authentication (Protocol 3.12)" />
  <figcaption aria-hidden="true">Example message flow of OAuth authentication (Protocol 3.12)</figcaption>
</figure>

#### (Protocol 3.15)

In OAuth 2.0, the term “grant type” refers to the way an application gets an access token. OAuth 2.0 defines several grant types, including the authorization code flow, which requires the app launch a browser to begin the flow.

New fields (Auth URL, Token URL, client_id) added in the [AuthenticationOAuth](#authenticationoauth-r) message to support OAuth browser workflow (i.e. Authorization Code grant type). 

At a high level, the authorization code flow has the following steps:

- The client application opens a browser to send the user to the OAuth server (Auth URL)
- The user sees the authorization prompt and approves the app’s request
- The user is redirected back to the client application with an authorization code in the query string
- The client application exchanges the authorization code (with Token URL) for an access token

*client_id* is required every time a HTTP request is issued to the IDP.
*client_secret* is required only for requests to the Token URL, but it is not allowed to be sent by the server for any reason. So the client application needs to provide the *client_secret* to the IDP to retrieve an access token.

<figure>
  <img src="/images/data-protocols-formats/frontent-backend/OAuth_browser_Msg_flow.png"                                   
       title="Example message flow of OAuth browser workflow (Protocol 3.15)"
       alt="Example message flow of OAuth browser workflow" />
  <figcaption aria-hidden="true">Example message flow of OAuth browser workflow (Protocol 3.15)</figcaption>
</figure>

#### (Protocol 3.16)
New fields (scope, validate_hostname) added in the [AuthenticationOAuth](#authenticationoauth-r) message.

### Simple Query

A simple query cycle is initiated by the frontend sending a [Query](#query-q) message to the backend. The message includes an SQL command (or commands) expressed as a text string. The backend then sends one or more response messages depending on the contents of the query command string, and finally a [ReadyForQuery](#readyforquery-z) response message. ReadyForQuery informs the frontend that it can safely send a new command.


![Example message flow of simple query with data rows returned](/images/data-protocols-formats/frontent-backend/SimpleQuery_Msg_flow.png "Example message flow of simple query with data rows returned")

The possible response messages from the backend are:

- [CommandComplete](#commandcomplete-c)
  : An SQL command completed normally.

- [RowDescription](#rowdescription-t)
  : Indicates that rows are about to be returned in response to a query. The contents of this message describe the column layout of the rows. This will be followed by a [DataRow](#datarow-d) message for each row being returned to the frontend.

- [VerifyFiles](#verifyfiles-f)
  : The backend starts processing a `COPY LOCAL` command; see Section [COPY Operations](#copy-operations) -- [Copy-Local mode](#copy-local-mode).

- [CopyInResponse](#copyinresponse-g)
  : The backend starts processing a `COPY STDIN` command; see Section [COPY Operations](#copy-operations) -- [Copy-Stdin mode](#copy-stdin-mode)

- [DataRow](#datarow-d)
  : One of the set of rows returned by a query.

- [EmptyQueryResponse](#emptyqueryresponse-i)
  : An empty query string was recognized.

- [ParameterStatus](#parameterstatus-s)
  : This message informs the frontend about the current setting of backend parameters, a `SET` query normally produce this message.

- [NoticeResponse](#noticeresponse-n)
  : A warning message has been issued in relation to the query. For example, issue a `COMMIT` when no transaction in progress. NoticeResponses are in addition to other responses, i.e., the backend will continue processing the command.

- [ErrorResponse](#errorresponse-e)
  : An error has occurred.

- [ReadyForQuery](#readyforquery-z)
  : Processing of the query string is complete. A separate message is sent to indicate this because the query string might contain multiple SQL commands. ([CommandComplete](#commandcomplete-c) marks the end of processing **one** SQL command, not the whole string.) ReadyForQuery will always be sent, whether processing terminates successfully or with an error.

The response to a `SELECT` query (or other queries that return row sets, such as `EXPLAIN` or `SHOW`) normally consists of [RowDescription](#rowdescription-t), zero or more [DataRow](#datarow-d) messages, and then [CommandComplete](#commandcomplete-c). `COPY` command invokes special protocol as described in Section [COPY Operations](#copy-operations). Other query types normally produce only a [CommandComplete](#commandcomplete-c) message.

Since a query string could contain several queries (separated by semicolons), there might be several such response sequences before the backend finishes processing the query string. ReadyForQuery is issued when the entire string has been processed and the backend is ready to accept a new query string.

If a completely empty (no contents other than whitespace) query string is received, the response is [EmptyQueryResponse](#emptyqueryresponse-i) followed by [ReadyForQuery](#readyforquery-z).

In the event of an error, [ErrorResponse](#errorresponse-e) is issued followed by ReadyForQuery. All further processing of the query string is aborted by ErrorResponse (even if more queries remained in it). Note that this might occur partway through the sequence of messages generated by an individual query.

Recommended practice is to code frontends in a state-machine style that will accept any message type at any time that it could make sense, rather than wiring in assumptions about the exact sequence of messages.

### Extended Query

The extended query protocol breaks down the above-described simple query protocol into multiple steps. The overall execution cycle consists of a ***parse* step**, which creates a prepared statement from a textual query string; a ***bind* step**, which creates a portal given a prepared statement and values for any needed parameters; and an ***execute* step** that runs a portal's query. One of the advantages of extended query protocol is preventing SQL injection attacks.

<figure>
  <img src="/images/data-protocols-formats/frontent-backend/PreparedStatement_Msg_flow.png"                                   
       title="Example message flow of extended query (Close message is not included)"
       alt="Example message flow of extended query (Close message is not included)" />
  <figcaption aria-hidden="true">Example message flow of extended query (Close message is not included)</figcaption>
</figure>

In the extended protocol, the frontend first sends a [Parse](#parse-p) message, which contains a textual query string. The query string leaves certain values (must be *SQL literals*; otherwise a syntax error is reported) unspecified with parameter placeholders (i.e. question mark '?'). The response is either [ParseComplete](#parsecomplete-1) or [ErrorResponse](#errorresponse-e).

> **Note**: The query string contained in a [Parse](#parse-p) message cannot include more than one SQL statement; else a syntax error is reported. This restriction does not exist in the [simple-query protocol](#simple-query).
> 
> The query string contained in a [Parse](#parse-p) message cannot be a `COPY FROM LOCAL` SQL statement; else a protocol error is reported. See Section [Copy-Local mode](#copy-local-mode).

If successfully created, a named prepared-statement object lasts till the end of the current session, unless explicitly destroyed. Within a session, the backend can keep track of multiple prepared statements. Existing prepared statements and portals are referenced by names assigned when they were created. The frontend can optimize its behavior when the same query string is used, but different parameters are bound to it (many times), i.e. *parse* once and then *bind* & *execute* many times.

Once a prepared statement exists, it can be readied for execution using a [Bind](#bind-b) message. The [Bind](#bind-b) message gives the name of the source prepared statement, the name of the destination portal (empty string denotes the unnamed portal), and the values to use for any parameter placeholders ('?') present in the prepared statement. The supplied parameter set must match those needed by the prepared statement.
[Bind](#bind-b) also specifies the format to use for any data returned by the query; the format can be specified overall, or per-column. The response is either [BindComplete](#bindcomplete-2) or ErrorResponse.

> **Note**: In Vertica server, there is only one portal. The portal name required in some messages is actually ignored by the backend.

Once a portal exists, it can be executed using an [Execute](#execute-e) message. The [Execute](#execute-e) message specifies the portal name (empty string denotes the unnamed portal) and a maximum result-row count (Vertica backend ignores this and always fetch all rows). The possible responses to [Execute](#execute-e) are the same as those described above for queries issued via simple query protocol, except that Execute doesn't cause ReadyForQuery or RowDescription to be issued, also [PortalSuspended](#portalsuspended-s) replaced [CommandComplete](#commandcomplete-c) to indicate completion of the source SQL command.

At completion of each series of extended-query messages, the frontend should issue a [Sync](#sync-s) message. This parameterless message causes the backend to close the current transaction if it's not inside a `BEGIN`/`COMMIT` transaction block ("close" meaning to commit if no error, or roll back if error). Then a ReadyForQuery response is issued. The purpose of [Sync](#sync-s) is to provide a resynchronization point for error recovery. When an error is detected while processing any extended-query message, the backend issues ErrorResponse, then reads and discards messages until a [Sync](#sync-s) is reached, then issues ReadyForQuery and returns to normal message processing. (But note that no skipping occurs if an error is detected while processing Sync — this ensures that there is one and only one ReadyForQuery sent for each Sync.)

In most scenarios the frontend should issue a [Describe](#describe-d) before issuing [Execute](#execute-e), to ensure that it knows how to interpret the results it will get back. The [Describe](#describe-d) message (statement variant) specifies the name of an existing prepared statement. The response is a [ParameterDescription](#parameterdescription-t) message describing the parameters needed by the statement, followed by a [RowDescription](#rowdescription-t) message describing the rows that will be returned when the statement is eventually executed (or a [NoData](#nodata-n) message if the statement will not return rows), and followed by a [CommandDescription](#commanddescription-m) message describing the type of command to be executed and any semantically-equivalent COPY statement. [ErrorResponse](#errorresponse-e) is issued if there is no such prepared statement. Note that since [Bind](#bind-b) has not yet been issued, the formats to be used for returned columns are not yet known to the backend; the format code fields in the [RowDescription](#rowdescription-t) message will be zeroes in this case. The [Describe](#describe-d) message (portal variant) specifies the name of an existing portal (or an empty string for the unnamed portal). The response is a [RowDescription](#rowdescription-t) message describing the rows that will be returned by executing the portal; or a [NoData](#nodata-n) message if the portal does not contain a query that will return rows; or [ErrorResponse](#errorresponse-e) if there is no such portal.

The [Close](#close-c) message closes an existing prepared statement or portal and releases resources. It is not an error to issue Close against a nonexistent statement or portal name. The response is normally [CloseComplete](#closecomplete-3), but could be [ErrorResponse](#errorresponse-e) if some difficulty is encountered while releasing resources. Note that closing a prepared statement implicitly closes any open portals that were constructed from that statement.

The [Flush](#flush-h) message does not cause any specific output to be generated, but forces the backend to deliver any data pending in its output buffers. A [Flush](#flush-h) must be sent after any extended-query command except [Sync](#sync-s), if the frontend wishes to examine the results of that command before issuing more commands. Without [Flush](#flush-h), messages returned by the backend will be combined into the minimum possible number of packets to minimize network overhead.

### COPY Operations

The `COPY` command allows high-speed bulk data transfer from the client to the server. COPY operations can be divided into [*copy-local* mode](#copy-local-mode) (it is further divided into *copy-local-stdin* mode and *copy-local-file* mode) and [*copy-stdin* mode](#copy-stdin-mode).

#### Copy-Local mode

Copy-local mode is initiated when the backend executes a `COPY FROM LOCAL FILES` or `COPY FROM LOCAL STDIN` SQL statement in [Query](#query-q) message. The backend sends a [RowDescription](#rowdescription-t) message (For the [DataRow](#datarow-d) message later indicating the number of rows loaded) and a [VerifyFiles](#verifyfiles-f) message to the frontend. The backend parses the file names out of the command, and sends them back to the frontend in [VerifyFiles](#verifyfiles-f) message. If `COPY` command uses the `REJECTED DATA` and/or `EXCEPTIONS` parameters, VerifyFiles message contains filenames for them. In *copy-local-file* mode, VerifyFiles message also contains input filenames. The frontend has to verify that these input files exist and are readable, and rejected data and exceptions files are writable.

Then the frontend should send a [VerifiedFile](#emptyqueryresponse-i) message specifying a list of input files that the backend can ask to load. In *copy-local-stdin* mode, this is an empty list. In *copy-local-file* mode, this must be a non-empty list, because sending an empty list of files will make server kill the session. If the backend does not send a ErrorResponse, it is ready to copy data from STDIN/files. The backend might send [ParameterStatus](#parameterstatus-s) messages, then

- In *copy-local-file* mode, for each file to load, the backend sends a [LoadFile](#loadfile-h) message to ask for data from a file. The frontend should then send zero or more [CopyData](#copydata-d) messages, forming a stream of input data. The message boundaries are not required to have anything to do with row boundaries, although that is often a reasonable choice. The frontend should send a [EndOfBatchRequest](#endofbatchrequest-j) message to indicate that a batch of rows has been sent, and the frontend is expecting an acknowledgment and possibly rejected row descriptions from the backend. The backend should then send zero or more [WriteFile](#writefile-o) messages for rejected row descriptions and a [EndOfBatchResponse](#endofbatchresponse-j) message for acknowledgement of data loading from this file. When the backend finishes asking for data from all files, it sends a [CopyDoneResponse](#copydoneresponse-c) message.

<figure>  
 <img src="/images/data-protocols-formats/frontent-backend/CopyLocalFile_Msg_flow.png"
      title="Example message flow of COPY FROM LOCAL FILE"
      width="500"
      alt="Example message flow of COPY FROM LOCAL FILE" />                   
      <figcaption aria-hidden="true">Example message flow of COPY FROM LOCAL FILE</figcaption>
 </figure> 

- In *copy-local-stdin* mode, the backend sends a [CopyInResponse](#copyinresponse-g) message to the frontend to ask for data. Same as in *copy-local-file* mode, the frontend should send zero or more [CopyData](#copydata-d) messages, forming a stream of input data, and a [EndOfBatchRequest](#endofbatchrequest-j) message and then receive zero or more [WriteFile](#writefile-o) messages and an [EndOfBatchRequest](#endofbatchrequest-j) message. In this mode, there will be no incoming message until the frontend send a [CopyDone](#copydone-c) message to end this STDIN loading.

<figure>
  <img src="/images/data-protocols-formats/frontent-backend/CopyLocalStdin_Msg_flow.png"
       title="Example message flow of COPY FROM LOCAL STDIN"
       width="500"
       alt="Example message flow of COPY FROM LOCAL STDIN"
  />
  <figcaption aria-hidden="true">Example message flow of COPY FROM LOCAL STDIN<figcaption>
</figure>


> **Security concerns**: The frontend must verify the file the backend asks for reading ([LoadFile](#loadfile-h)) or writing ([WriteFile](#writefile-o)) is not tampered.

After loading all the data, the backend sends a [DataRow](#datarow-d) message indicating the number of rows loaded, and then a [CommandComplete](#commandcomplete-c) message indicating the end of the `COPY` command. The query string might contain multiple SQL commands, the backend will return to the command-processing mode of [Simple Query protocol](#simple-query) when a `COPY` command ended. ReadyForQuery will always be sent when the query string is complete.

In the event of a frontend-detected error during *copy-local* mode, the frontend can terminate it by sending a [CopyError](#copyerror-e) message, which will cause the `COPY` SQL statement to fail with an ErrorResponse. In the event of a backend-detected error (including receipt of a CopyError message), the backend will issue an ErrorResponse message, any subsequent messages issued by the frontend will simply be dropped, and ReadyForQuery is issued.

#### Copy-Stdin mode

This protocol comes from PostgreSQL and is (partially) supported in Vertica server but not implemented by most Vertica clients. The backend cannot report the number of rows loaded after copy, and possibly rejected row descriptions.

Copy-stdin mode is initiated when the backend executes a `COPY FROM STDIN` SQL statement (**not** "COPY FROM LOCAL STDIN") in a [Query](#query-q) message. The backend might send [ParameterStatus](#parameterstatus-s) messages, and then a [CopyInResponse](#copyinresponse-g) message to the frontend to ask for data. The frontend should send zero or more [CopyData](#copydata-d) messages, forming a stream of input data, and a [CopyDone](#copydone-c) message to end this STDIN loading. The backend should process the command and send a [CommandComplete](#commandcomplete-c) message indicating the end of the `COPY` command. The query string might contain multiple SQL commands, the backend will return to the command-processing mode of [Simple Query protocol](#simple-query) when a `COPY` command ended. ReadyForQuery will always be sent when the query string is complete.

![Example message flow of COPY FROM STDIN](/images/data-protocols-formats/frontent-backend/CopyStdin_Msg_flow.png "Example message flow of COPY FROM STDIN")

In the event of a frontend-detected error during *copy-stdin* mode, the frontend can terminate it by sending a [CopyFail](#copyfail-f) or a [CopyError](#copyerror-e) message, which will cause the `COPY` SQL statement to fail with an ErrorResponse. In the event of a backend-detected error (including receipt of a [CopyFail](#copyfail-f) or a [CopyError](#copyerror-e) message), the backend will issue an ErrorResponse message, any subsequent CopyData, CopyDone, CopyFail or CopyError messages issued by the frontend will simply be dropped, and ReadyForQuery is issued.

Copy-stdin mode can also be initiated via [Extended Query protocol](#extended-query), the `COPY` command is issued in a [Parse](#parse-p) message. The workflow is the same as normal processing of [Extended Query protocol](#extended-query) except that the backend should send a [CopyInResponse](#copyinresponse-g) message after receiving an [Execute](#execute-e) message. Then the frontend should send zero or more [CopyData](#copydata-d) messages, forming a stream of input data, and a [CopyDone](#copydone-c) or [CopyFail](#copyfail-f)/[CopyError](#copyerror-e) message to end the STDIN loading.

### Asynchronous Operations

There are several cases in which the backend will send messages that are not specifically prompted by the frontend's command stream. Frontends must be prepared to deal with these messages at any time, even when not engaged in a query.

[NoticeResponse](#noticeresponse-n)
: It is possible for NoticeResponse messages to be generated due to outside activity; for example, if the database administrator commands a "fast" database shutdown, the backend will send a NoticeResponse indicating this fact before closing the connection. Accordingly, frontends should always be prepared to accept and display NoticeResponse messages, even when the connection is nominally idle.

[ParameterStatus](#parameterstatus-s)
: ParameterStatus messages will be generated whenever the active value changes for any of the parameters the backend believes the frontend should know about. For example, when you do `SET SESSION AUTOCOMMIT ON | OFF`, you get back a ParameterStatus telling you the new value of autocommit.
: At present Vertica supports a handful of parameters for which ParameterStatus will be generated, they are: *standard_conforming_strings*, *server_version*, *client_locale*, *client_label*, *long_string_types*, *protocol_version*, *auto_commit*, *mars*, *timezone*, *database_name*, *request_complex_types*, *extend_copy_reject_info* etc.
: More parameters might be added in the future. A frontend should simply ignore ParameterStatus for parameters that it does not understand or care about.

### Canceling Requests in Progress

During the processing of a query, the frontend might request cancellation of the query. The cancel request is not sent directly on the open connection to the backend for reasons of implementation efficiency: we don't want to have the backend constantly checking for new input from the frontend during query processing. Cancel requests should be relatively infrequent, so we make them slightly cumbersome in order to avoid a penalty in the normal case.

To issue a cancel request, the frontend opens a new connection to the server and sends a [CancelRequest](#cancelrequest) message, rather than the [StartupRequest](#startuprequest) message that would ordinarily be sent across a new connection. The server will process this request and then close the connection. For security reasons, no direct reply is made to the [CancelRequest](#cancelrequest) message.

A [CancelRequest](#cancelrequest) message will be ignored unless it contains the same key data (PID and secret key) passed to the frontend during connection start-up. If the request matches the PID and secret key for a currently executing backend, the processing of the current query is aborted.

The cancellation signal might or might not have any effect — for example, if it arrives after the backend has finished processing the query, then it will have no effect. If the cancellation is effective, it results in the current command being terminated early with an error message saying "Execution canceled by operator".

The upshot of all this is that for reasons of both security and efficiency, the frontend has no direct way to tell whether a cancel request has succeeded. It must continue to wait for the backend to respond to the query. Issuing a cancel simply improves the odds that the current query will finish soon, and improves the odds that it will fail with an error message instead of succeeding.

Since the cancel request is sent across a new connection to the server and not across the regular frontend/backend communication link, it is possible for the cancel request to be issued by any process, not just the frontend whose query is to be canceled. This might provide additional flexibility when building multiple-process applications. It also introduces a security risk, in that unauthorized persons might try to cancel queries. The security risk is addressed by requiring a dynamically generated secret key to be supplied in [CancelRequest](#cancelrequest).

### Termination

The normal, graceful termination procedure is that the frontend sends a [Terminate](#terminate-x) message and immediately closes the connection. On receipt of this message, the backend closes the connection and terminates.

In rare cases (such as an administrator-commanded database shutdown) the backend might disconnect without any frontend request to do so. In such cases the backend will attempt to send an [ErrorResponse](#errorresponse-e) or [NoticeResponse](#noticeresponse-n) message giving the reason for the disconnection before it closes the connection.

## Message Data Types

This section describes the base data types used in messages.

Int<strong><i>n</i></strong>(<strong><i>i</i></strong>)
: An <strong><i>n</i></strong>-bit integer in network byte order (most significant byte first). If <strong><i>i</i></strong> is specified it is the exact value that will appear, otherwise the value is variable. Eg. Int16, Int32(42).

Int<strong><i>n</i></strong>[<strong><i>k</i></strong>]
: An array of <strong><i>k</i></strong> <strong><i>n</i></strong>-bit integers, each in network byte order. The array length <strong><i>k</i></strong> is always determined by an earlier field in the message. Eg. Int16\[M\].

String(<strong><i>s</i></strong>)
: A null-terminated string (C-style string). There is no specific length limitation on strings. If <strong><i>s</i></strong> is specified it is the exact value that will appear, otherwise the value is variable. Eg. String, String("user").

 > Note: There is no predefined limit on the length of a string that can be returned by the backend. Good coding strategy for a frontend is to use an expandable buffer so that anything that fits in memory can be accepted. If that's not feasible, read the full string and discard trailing characters that don't fit into your fixed-size buffer.

String[<strong><i>k</i></strong>]
: An array of <strong><i>k</i></strong> null-terminated strings (C-style strings). The array length <strong><i>k</i></strong> is always determined by an earlier field in the message. Eg. String\[N\]. 

Byte<strong><i>n</i></strong>(<strong><i>c</i></strong>)
: Exactly <strong><i>n</i></strong> bytes. If the field width <strong><i>n</i></strong> is not a constant, it is always determinable from an earlier field in the message. If <strong><i>c</i></strong> is specified it is the exact value. Eg. Byte2, Byte1('\n').

## Message Formats

This section describes the detailed format of each message. Each message is classified to indicate that it may be sent by a frontend (client), or a backend (server). In most cases, the first byte of a message identifies the message type, and the next four bytes specify the length of the rest of the message (this length count includes itself, but not the message-type byte). The remaining contents of the message are determined by the message type. There are a few message types that have no initial message-type byte.

### Frontend

#### Bind 'B'

| Type       | Description |
|:-----------|:------------|
| Byte1('B') | Identifies the message as a Bind command. |
| Int32      | Length of message contents in bytes, including self. |
| String     | The name of the destination portal (an empty string selects the unnamed portal). |
| String     | The name of the source prepared statement (an empty string selects the unnamed prepared statement). |
| Int16      | The number of parameter format codes that follow (denoted ***C*** below). This can be zero to indicate that there are no parameters or that the parameters all use the default format (text); or one, in which case the specified format code is applied to all parameters; or it can equal the actual number of parameters. |
| Int16\[***C***\] | The parameter format codes. Each must presently be zero (text) or one (binary). |
| Int16      | The number of parameter values that follow (possibly zero) (denoted ***P*** below). This must match the number of parameters needed by the query. |
| Int32\[***P***\] | The parameter type Oids. |
| Next, the following pair of fields appear for each parameter: <td colspan=2> | |
| Int32     | The length of the parameter value, in bytes (this count does not include itself) (denoted ***n*** below). Can be zero. As a special case, -1 indicates a *NULL* parameter value. No value bytes follow in the *NULL* case. |
| Byte***n*** | The value of the parameter, in the format indicated by the associated format code. |
| After the last parameter, the following fields appear:        |  |
| Int16       | The number of result-column format codes that follow (denoted ***R*** below). This can be zero to indicate that there are no result columns or that the result columns should all use the default format (text); or one, in which case the specified format code is applied to all result columns (if any); or it can equal the actual number of result columns of the query. |
| Int16\[***R***\] | The result-column format codes. Each must presently be zero (text) or one (binary).  |

#### CancelRequest

| Type       | Description |
|:-----------|:------------|
| Int32(16)  | Length of message contents in bytes, including self.  |
| Int32(80877102) | The cancel request code. The value is chosen to contain *1234* in the most significant 16 bits, and *5678* in the least 16 significant bits. (To avoid confusion, this code must not be the same as any protocol version number.) |
| Int32           | The process ID of the target backend.  |
| Int32           | The secret key for the target backend. |

#### ChangePassword 'n'

| Type       | Description |
|:-----------|:------------|
| Byte1('n') | Identifies the message as a ChangePassword command.  |
| Int32      | Length of message contents in bytes, including self. |
| String     | The new password. |

#### Close 'C'

| Type       | Description |
|:-----------|:------------|
| Byte1('C') | Identifies the message as a Close command. |
| Int32      | Length of message contents in bytes, including self. |
| Byte1      | 'S' to close a prepared statement; or 'P' to close a portal. |
| String     | The name of the prepared statement or portal to close (an empty string selects the unnamed prepared statement or portal). |

#### CopyData 'd'

| Type       | Description |
|:-----------|:------------|
| Byte1('d')  | Identifies the message as COPY data. |
| Int32       | Length of message contents in bytes, including self. |
| Byte***n*** | Data that forms part of a COPY data stream. Messages sent by frontends may divide the data stream arbitrarily. |

#### CopyDone 'c'

| Type       | Description |
|:-----------|:------------|
| Byte1('c') | Identifies the message as a COPY-complete indicator.<br>Note: This is different from a [CopyDoneResponse](#copydoneresponse-c) backend message. |
| Int32(4)   | Length of message contents in bytes, including self. |

#### CopyError 'e'
| Type       | Description |
|:-----------|:------------|
| Byte1('e') | Identifies the message as a copy error. Terminate the Copy-Local protocol. |
| Int32      | Length of message contents in bytes, including self.                       |
| String     | Error file name.                                                           |
| Int32      | Error line number.                                                         |
| String     | Error method name.                                                         |
| String     | Error message.                                                             |

#### CopyFail 'f'

| Type       | Description |
|:-----------|:------------|
| Byte1('f') | Identifies the message as a COPY-failure indicator.  |
| Int32      | Length of message contents in bytes, including self. |
| String     | An error message to report as the cause of failure.  |

#### Describe 'D'

| Type       | Description |
|:-----------|:------------|
| Byte1('D') | Identifies the message as a Describe command.                                                                                |
| Int32      | Length of message contents in bytes, including self.                                                                         |
| Byte1      | 'S' to describe a prepared statement; or 'P' to describe a portal.                                                           |
| String     | The name of the prepared statement or portal to describe (an empty string selects the unnamed prepared statement or portal). |

#### EndOfBatchRequest 'j'

| Type       | Description |
|:-----------|:------------|
| Byte1('j') | Identifies the message as a EndOfBatchRequest command. Signals that a batch of rows has been sent, and the client is expecting an acknowledgment and possibly rejected row descriptions from the server. |
| Int32(4)   | Length of message contents in bytes, including self. |

#### Execute 'E'

| Type       | Description |
|:-----------|:------------|
| Byte1('E') | Identifies the message as an Execute command.  |
| Int32      | Length of message contents in bytes, including self.  |
| String     | The name of the portal to execute (an empty string selects the unnamed portal). |
| Int32      | Maximum number of rows to return, if portal contains a query that returns rows (ignored otherwise). Zero denotes "no limit".<br>***Note***: Currently, Vertica backend will ignore this result-row count and send all the rows regardless of what you put here. |

#### Flush 'H'

| Type       | Description |
|:-----------|:------------|
| Byte1('H') | Identifies the message as a Flush command.           |
| Int32(4)   | Length of message contents in bytes, including self. |

#### LoadBalanceRequest

| Type       | Description |
|:-----------|:------------|
| Int32(8)        | Length of message contents in bytes, including self.  |
| Int32(80936960) | The LoadBalance request code. The value is chosen to contain *1235* in the most significant 16 bits, and *0000* in the least 16 significant bits. (To avoid confusion, this code must not be the same as any protocol version number.) |

#### MarsRequest '\_'

| Type        | Description |
|:------------|:------------|
| Byte1('\_') | Identifies the message as a MARS request.            |
| Int32(20)   | Length of message contents in bytes, including self. |
| Int32       | ResultSet ID                                         |
| Int32       | RequestType: Fetch(0x1) Close(0x2) NoRowDesc(0x4)    |
| Int64       | FetchCount                                           |

#### Parse 'P'

| Type       | Description |
|:-----------|:------------|
| Byte1('P') | Identifies the message as a Parse command. |
| Int32      | Length of message contents in bytes, including self.  |
| String     | The name of the destination prepared statement (an empty string selects the unnamed portal, i.e. prepared statement must be named). |
| String     | The query string to be parsed.  |
| Int16      | The number of parameter data types specified (may be zero). Note that this is not an indication of the number of parameters that might appear in the query string, only the number that the frontend wants to pre-specify types for. |
| Then, for each parameter, there is the following: | |
| Int32      | Specifies the object ID of the parameter data type. Placing a zero here is equivalent to leaving the type unspecified. The server will ignore these values. |                                                                                                           |

#### Password 'p'

| Type       | Description |
|:-----------|:------------|
| Byte1('p') | Identifies the message as a password. |
| Int32      | Length of message contents in bytes, including self. |
| String     | The password (encrypted, if requested). All password strings processed by Vertica require a null terminator, as is standard with any string in the Front End / Back End protocol.<br>GSS is special because Vertica doesn't touch the buffer it gets -- it defers to the GSS library instead. It is an error to send an extra null terminator in a GSS token message. |

#### Query 'Q'

| Type       | Description |
|:-----------|:------------|
| Byte1('Q') | Identifies the message as a simple query.            |
| Int32      | Length of message contents in bytes, including self. |
| String     | The query string itself.  |

#### SSLRequest

| Type       | Description |
|:-----------|:------------|
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(80877103) | The SSL request code. The value is chosen to contain *1234* in the most significant 16 bits, and *5679* in the least 16 significant bits. (To avoid confusion, this code must not be the same as any protocol version number.) |

#### StartupRequest

<table>
   <thead>
      <tr>
         <th>
            <p>Type</p>
         </th>
         <th>
            <p>Description</p>
         </th>
      </tr>
   </thead>
   <tbody>
      <tr>
         <td>
            <p>Int32</p>
         </td>
         <td>
            <p>Length of message contents in bytes, including self.</p>
         </td>
      </tr>
      <tr>
         <td>
            <p>Int32</p>
         </td>
         <td>
            <p>The <strong>fixed</strong> protocol version number. The most significant 16 bits are the major version number. The least significant 16 bits are the minor version number. For example, protocol version number 3.8 should be described as (3 &lt;&lt; 16 | 8) = 196616.</p>
            <p>Frontend and backend must negotiate and use the same protocol version for communication. For historical reasons, in this message, there are two protocol version numbers send to the backend: <strong>fixed</strong> protocol version and <strong>requested</strong> protocol version. The behavior depends on the protocol version of the backend:</p>
            <ul>
               <li>New servers (protocol version &gt;= 3.7)</li>
            </ul>
            <dl>
               <dt></dt>
               <dd>
                  <dl>
                     <dt></dt>
                     <dd>
                        The backend would use the frontend's requested protocol version to find
                        the highest common protocol version in use for both frontend and
                        backend, and send this <strong>effective</strong> protocol version back in a <a
                           href="#parameterstatus-s">ParameterStatus</a> message, with parameter name
                        'protocol_version'. The frontend can use this value to enable/disable
                        functionalities.
                     </dd>
                  </dl>
               </dd>
            </dl>
            <ul>
               <li>Old servers (protocol version &lt; 3.7)</li>
            </ul>
            <dl>
               <dt></dt>
               <dd>
                  <dl>
                     <dt></dt>
                     <dd>
                        The backend ignores the requested version and only looks at the fixed
                        protocol version. The backend accepts the connection if frontend's
                        protocol version is equal or older than the backend's.
                     </dd>
                  </dl>
               </dd>
            </dl>
         </td>
      </tr>
      <tr>
         <td colspan="2">
            <p>Next, the following pair of fields appears for each parameter. Parameters can appear in any order.</p>
         </td>
         <td></td>
      </tr>
      <tr>
         <td>
            <p>String</p>
         </td>
         <td>
            <p>The parameter name. The following table contains the definition
               for currently recognized parameter names:
            </p>
            <table>
               <thead>
                  <tr>
                     <th>
                        <p>Parameter name</p>
                     </th>
                     <th>
                        <p>Description</p>
                     </th>
                  </tr>
               </thead>
               <tbody>
                  <tr>
                     <td>user</td>
                     <td>
                        <p>The database user name to connect as. This is required unless a value is supplied for oauth_access_token, in which case the user can be an empty string. Otherwise it is required. There is no default.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>
                        <p>database</p>
                     </td>
                     <td>
                        <p>The database to connect to.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>protocol_version</td>
                     <td>
                        <p>The <strong>requested</strong> protocol version number, which is the highest version the frontend supports.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>binary_data_protocol</td>
                     <td>
                        <p>'0' is the default. If set to '1', enable this session-level setting. For additional details, see <a href="#formats-formatcodes">Formats and Format Codes</a>.</p>
                     </td>
                  </tr>
                  <tr>
                     <tdr>
                     <td>
                        <p>autocommit</p>
                     </td>
                     <td>
                        <p>'off' is the default. If set to 'on', enable AUTOCOMMIT setting.</p>
                     </td>
                  </tr>
                  <tr>
                     <tdtd>
                  </tr>
                  <td>client_label</td>
                  <td>
                     <p>The client connection label. You can use this label to distinguish client connections.</p>
                  </td>
                  </tr>
                  <tr>
                     <td>client_type</td>
                     <td>
                        <p>A string distinguish the type of this client. E.g. "JDBC Driver", "ADO.NET Driver".</p>
                     </td>
                  </tr>
                  <tr>
                     <td>client_version</td>
                     <td>
                        <p>A string distinguish the version of this client. Note that this is not required to follow the Vertica version number scheme. The open source drivers vertica-python and vertica-sql-go have independent version numbers.</p>
                     </td>
                  </tr>
                  <tr>
                     <tdr>
                     <td>
                        <p>client_pid</p>
                     </td>
                     <td>
                        <p>A string distinguish the process ID of this client.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>client_os</td>
                     <td>
                        <p>A string distinguish the client's OS platform.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>client_os_user_name</td>
                     <td>
                        <p>A string distinguish the OS login username of this client.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>client_os_hostname</td>
                     <td>
                        <p>A string distinguish the OS hostname of this client.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>mars</td>
                     <td>
                        <p>'off' is the default. If set to 'on', enable MARS setting.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>oauth_access_token</td>
                     <td>
                        <p>An OAuth token that authorizes a user to the database. Specifying a value triggers <a
                           href="#oauth2-authentication">OAuth authentication</a>.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>auth_category</td>
                     <td>
                        <p>A string indicating the type of authentication the client is prepared to do. Recognized values are "User", "Kerberos" and "OAuth". Specify only one type at a time.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>protocol_features</td>
                     <td>
                        <p>A JSON string of features requested by the driver. All features are disabled by default. Each feature enabled by the server will return a ParameterStatus message. These features are not bind with specific protocol version range. Currently supports the following features:</p>
                        <table>
                           <thead>
                            <tr>
                              <th>
                                <p>Feature name</p>
                              </th>
                              <th>
                                <p>Description</p>
                              </th>
                              <th>
                                <p>Support since</p>
                              </th>
                            </tr>
                           </thead>
                           <tbody>
                            <tr>
                              <td>request_complex_types</td>
                              <td>If set to <em>true</em>, server will return complex type metadata. Otherwise, complex types will be treated as long varchar.</td>
                              <td>Server v12.0SP2</td>
                            </tr>
                            <tr>
                              <td>extend_copy_reject_info</td>
                              <td>If set to <em>true</em>, server will return a modified format of the <a href="#writefile-o">WriteFile</a> message when the RETURNREJECTED keyword is used in a copy command. This allows full diagnostic results from a failed copy command to be identified row by row.</td>
                              <td>Server v23.3.0</td>
                            </tr>
                           </tbody>
                        </table>
                        <p>Example value: {"request_complex_types":true,"extend_copy_reject_info":false}</p>
                     </td>
                  </tr>
                  <tr>
                     <td>protocol_compat</td>
                     <td>
                        <p>A string representing the protocol that the driver is able to understand. Used in postgres compatibility. The server will try and determine the type of driver and which protocol to be used. If this parameter is provided, the server will defer to the value given instead of determining which protocol to use on its own. This may be used in conversion and extension of PG drivers to understand parts of the Vertica protocol.<br>
Currently recognized values for protocol_compat are "PG" or "VER" for Postgres and Vertica respectively. If no parameter is passed, the server will attempt to determine the value based on other parameters provided.</p>
                     </td>
                  </tr>
                  <tr>
                     <td>workload</td>
                     <td>
                        <p>A string representing the workload name to be used by workload routing rules.</p>
                     </td>
                  </tr>
               </tbody>
            </table>
         </td>
      </tr>
      <tr>
         <td>String</td>
         <td>The parameter value. Most parameter values are of type String
            except for the <strong>protocol_version</strong> parameter. It is of
            type Int32 followed by an extra null terminator ('\0').</p>
         </td>
      </tr>
      <tr>
         <td colspan="2">
            <p>And finally:</p>
         </td>
         <td></td>
      </tr>
      <tr>
         <td>Byte1('\0')</td>
         <td>
            <p>A zero byte is required as a terminator after the last name/value
               pair.
            </p>
         </td>
      </tr>
</table>

#### Sync 'S'

| Type       | Description |
|:-----------|:------------|
| Byte1('S') | Identifies the message as a Sync command. |
| Int32(4)   | Iength of message contents in bytes, including self. |

#### Terminate 'X'

| Type       | Description |
|:-----------|:------------|
| Byte1('X') | Identifies the message as a termination. |
| Int32(4)   | Length of message contents in bytes, including self. |

#### VerifiedFiles 'F'

| Type       | Description |
|:-----------|:------------|
| Byte1('F') | Identifies the message as a response to [VerifyFiles 'F'](#verifyfiles-f) message. Provide a list of files that the backend can ask to load. |
| Int32      | Length of message contents in bytes, including self.  |
| Int32      | The number of files. 0 if copy input is STDIN.<br>Note: Before v3.15 this is a Int16 type.  |
| Next, the following pair of fields appear for each file: |  |
| String     | File name. The string is null terminated. |
| Int64  | File content length in bytes.  |

### Backend

#### AuthenticationOk 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(0)   | Specifies that the authentication was successful.    |

#### AuthenticationCleartextPassword 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(3)   | Specifies that a clear-text password is required.    |

#### AuthenticationMD5Password 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(32)  | Length of message contents in bytes, including self. |
| Int32(5)   | Specifies that an MD5-encrypted password is required.<br>*password* = MD5(MD5(password + user) + salt) |
| Byte4      | The salt to use when encrypting the password. |
| Int32      | The user salt length, must equal to 16. |
| Byte16     | The user salt, but not used. |

#### AuthenticationGSS 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(7)   | Specifies that GSS authentication is required. |

#### AuthenticationGSSContinue 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32      | Length of message contents in bytes, including self. |
| Int32(8)   | Specifies that GSS authentication continue is required. |
| Byte**n**  | GSS data, which should be passed into an authGSSClientStep. |

#### AuthenticationPasswordExpired 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R')     | Identifies the message as an authentication request. |
| Int32          | Length of message contents in bytes, including self. |
| Int32(9)       | Specifies that the password is expired. |
| Int32\[**n**\] | A bunch of integers that represent some restrictions on the password (min/max length, character classes, etc). |

#### AuthenticationPasswordChanged 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(10)  | Specifies that the password is changed. |

#### AuthenticationPasswordGrace 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(11)  | Specifies that the password is in its grace period.  |

#### AuthenticationOAuth 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R') | Identifies the message as an authentication request. |
| Int32(8)   | Length of message contents in bytes, including self. |
| Int32(12)  | Specifies that an OAuth access token is required.    |
| String     | OAuth Auth URL. <em>New in version 3.15</em> |
| String     | OAuth Token URL. <em>New in version 3.15</em> |
| String     | OAuth Client ID. <em>New in version 3.15</em> | 
| String     | OAuth Scope. <em>New in version 3.16</em> |
| String     | OAuth validate hostname. <em>New in version 3.16</em> |

#### AuthenticationHashPassword 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R')   | Identifies the message as an authentication request. |
| Int32(32)    | Length of message contents in bytes, including self. |
| Int32(65536) | Specifies that a hashed password is required.<br>*password* = SHA512(SHA512(password + userSalt) + salt) |
| Byte4        | The salt to use when encrypting the password. |
| Int32        | The user salt length, must equal to 16. |
| Byte16       | The user salt. |

#### AuthenticationHashMD5Password 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R')   | Identifies the message as an authentication request. |
| Int32(32)    | Length of message contents in bytes, including self. |
| Int32(65541) | Specifies that a hashed MD5-encrypted password is required.<br>*password* = MD5(MD5(password + user) + salt) |
| Byte4        | The salt to use when encrypting the password. |
| Int32        | The user salt length, must equal to 16. |
| Byte16       | The user salt, but not used. |

#### AuthenticationHashSHA512Password 'R'

| Type       | Description |
|:-----------|:------------|
| Byte1('R')   | Identifies the message as an authentication request. |
| Int32(32)    | Length of message contents in bytes, including self. |
| Int32(66048) | Specifies that a hashed SHA512-encrypted password is required.<br>*password* = SHA512(SHA512(password + userSalt) + salt) |
| Byte4        | The salt to use when encrypting the password. |
| Int32        | The user salt length, must equal to 16. |
| Byte16       | The user salt. |

#### BackendKeyData 'K'

| Type       | Description |
|:-----------|:------------|
| Byte1('K') | Identifies the message as cancellation key data. The frontend must save these values if it wishes to be able to issue [CancelRequest](#cancelrequest) messages later. |
| Int32(12)  | Length of message contents in bytes, including self.  |
| Int32      | The process ID of this backend.  |
| Int32      | The secret key of this backend.  |  

#### BindComplete '2'

| Type       | Description |
|:-----------|:------------|
| Byte1('2') | Identifies the message as a Bind-complete indicator. |
| Int32(4)   | Length of message contents in bytes, including self. |

#### CloseComplete '3'

| Type       | Description |
|:-----------|:------------|
| Byte1('3') | Identifies the message as a Close-complete indicator. |
| Int32(4)   | Length of message contents in bytes, including self.  |

#### CommandComplete 'C'

| Type       | Description |
|:-----------|:------------|
| Byte1('C') | Identifies the message as a command-completed response.                                    |
| Int32      | Length of message contents in bytes, including self.                                       |
| String     | The command tag. This is usually a single word that identifies which SQL command was sent. |

#### CommandDescription 'm'

| Type       | Description |
|:-----------|:------------|
| Byte1('m') | Identifies the message as a command description.                                                 |
| Int32      | Length of message contents in bytes, including self.                                             |
| String     | The command tag. This is usually a single word that identifies which SQL command was sent.       |
| Int16      | 1 if command was a prepared INSERT that can be converted to a COPY STDIN statement; 0 otherwise. |
| String     | The COPY STDIN statement if it was possible to convert the prepared INSERT into a COPY.          |

#### CopyDoneResponse 'c'

| Type       | Description |
|:-----------|:------------|
| Byte1('c') | Identifies the message as a Copy done response for "COPY FROM LOCAL FILES" command.<br>Note: This is different from a [CopyDone](#copydone-c) frontend message. |
| Int32(4)   | Length of message contents in bytes, including self. |

#### CopyInResponse 'G'

| Type       | Description |
|:-----------|:------------|
| Byte1('G') | Identifies the message as a Start Copy In response.  |
| Int32            | Length of message contents in bytes, including self. |
| Int8             | 0 indicates the overall COPY format is textual (rows separated by newlines, columns separated by separator characters, etc). 1 indicates the overall copy format is binary (similar to DataRow format). |
| Int16            | The number of columns in the data to be copied (denoted ***N*** below). |
| Int16\[***N***\] | The format codes to be used for each column. Each must presently be zero (text) or one (binary). All must be zero if the overall copy format is textual. |

#### DataRow 'D'

| Type       | Description |
|:-----------|:------------|
| Byte1('D') | Identifies the message as a data row.  |
| Int32      | Length of message contents in bytes, including self. |
| Int16      | The number of column values that follow (possibly zero). |
| Next, the following pair of fields appear for each column: | |
| Int32     | The length of the column value, in bytes (this count does not include itself) (denoted ***n*** below). Can be zero. As a special case, -1 indicates a *NULL* column value. No value bytes follow in the *NULL* case. |
| Byte***n*** | The value of the column, in the format indicated by the associated [format code](#formats-and-format-codes). |

#### EmptyQueryResponse 'I'

| Type       | Description |
|:-----------|:------------|
| Byte1('I') | Identifies the message as a response to an empty query string. This substitutes for [CommandComplete](#commandcomplete-c) message. |
| Int32(4)   | Length of message contents in bytes, including self. |

#### EndOfBatchResponse 'J'

| Type       | Description |
|:-----------|:------------|
| Byte1('J') | Signals that the server is done sending back rejected rows, and is ready for the next set of rows to be sent (or the copy end message). |
| Int32(4)   | Length of message contents in bytes, including self.  |

#### ErrorResponse 'E'

| Type       | Description |
|:-----------|:------------|
| Byte1('E') | Identifies the message as an error.  |
| Int32      | Length of message contents in bytes, including self. |
| The message body consists of one or more identified fields, followed by a zero byte as a terminator. Fields may appear in any order. For each field there is the following: |   |
| Byte1      | A code identifying the field type; if zero, this is the message terminator and no string follows. The presently defined field types are listed in [Error and Notice Message Fields](#error-and-notice-message-fields) section. Since more field types may be added in future, frontends should silently ignore fields of unrecognized type. |
| String  | The field value. |

#### LoadBalanceResponse 'Y/N'

| Type       | Description |
|:-----------|:------------|
| Byte1('N') | Identifies the message as a LoadBalance response. The backend <u>rejects</u> the frontend LoadBalance request.<br>Note: This is different from a [NoticeResponse 'N'](#noticeresponse-n) message. |

or

| Type       | Description |
|:-----------|:------------|
| Byte1('Y') | Identifies the message as a LoadBalance response. The backend <u>accepts</u> the frontend LoadBalance request. |
| Int32      | Length of message contents in bytes, including self. |
| Int32      | The port number of the load balance target. |
| String     | The host of the load balance target. The string is null terminated. This is an IP address, not a DNS name. |

#### LoadFile 'H'

| Type       | Description |
|:-----------|:------------|
| Byte1('H') | Identifies the message as a Copy-Local response to load the data from a file. |
| Int32      | Length of message contents in bytes, including self.                          |
| String     | The name of the file to load.                                                 |

#### MarsResponse '\_'

| Type       | Description |
|:-----------|:------------|
| Byte1('\_') | Identifies the message as a MARS response.                              |
| Int32(20)   | Length of message contents in bytes, including self.                    |
| Int32       | ResultSet ID.                                                           |
| Int32       | Status: Created(0x1) Fetched(0x2) Closed(0x4) Canceled(0x8) Error(0x10) |
| Int64       | Remaining rowcount.                                                     |

#### NoData 'n'

| Type       | Description |
|:-----------|:------------|
| Byte1('n') | Identifies the message as a no-data indicator. It is a response for transactions/DDL. |
| Int32(4)   | Length of message contents in bytes, including self.                                  |

#### NoticeResponse 'N'

| Type       | Description |
|:-----------|:------------|
| Byte1('N') | Identifies the message as a notice. Note: This is different from a [LoadBalanceResponse 'N'](#loadbalanceresponse-yn) message.  |
| Int32  | Length of message contents in bytes, including self.  |
| The message body consists of one or more identified fields, followed by a zero byte as a terminator. Fields may appear in any order. For each field there is the following: |  |
| Byte1  | A code identifying the field type; if zero, this is the message terminator and no string follows. The presently defined field types are listed in [Error and Notice Message Fields](#error-and-notice-message-fields) section. Since more field types may be added in future, frontends should silently ignore fields of unrecognized type. |
| String  | The field value. |

#### ParameterDescription 't'

| Type       | Description |
|:-----------|:------------|
| Byte1('t') | Identifies the message as a parameter description. |
| Int32      | Length of message contents in bytes, including self.                                                                                                                                    |
| Int16      | The number of parameters used by the statement (may be zero).                                                                                                                           |
| Int32      | The number of elements in the type mapping pool, which is an array. This pool is used for non-native types like GEOMETRY and GEOGRAPHY (They are backed by LONG VARBINARY native type). |
| Then, for each type mapping element in the pool, there is the following: |  |
| Int32      | Specifies the base object ID of the non-native data type.  |
| String     | Specifies the name of the non-native data type. |
| Then, for each parameter, there is the following:  |
| Byte1      | '1' if the parameter uses the type mapping pool; '0' otherwise.   |
| Int32      | If the parameter uses the type mapping pool, specifies the position (0-indexed) in the type mapping pool. Otherwise, specifies the object ID of the parameter data type. |
| Int32      | Specifies the parameter type modifier.    |
| Int16      | 1 if the column has a *NOT NULL* constraint; 0 otherwise.    |

<a id="parameterstatus"></a>
#### ParameterStatus 'S'

| Type       | Description |
|:-----------|:------------|
| Byte1('S') | Identifies the message as a run-time parameter status report. |
| Int32      | Length of message contents in bytes, including self.          |
| String     | The name of the run-time parameter being reported.            |
| String     | The current value of the parameter.                           |

#### ParseComplete '1'

| Type       | Description |
|:-----------|:------------|
| Byte1('1') | Identifies the message as a Parse-complete indicator. |
| Int32(4)   | Length of message contents in bytes, including self.  |

#### PortalSuspended 's'

| Type       | Description |
|:-----------|:------------|
| Byte1('s') | Indicates that a portal has stopped execution.       |
| Int32(4)   | Length of message contents in bytes, including self. |

#### ReadyForQuery 'Z'

| Type       | Description |
|:-----------|:------------|
| Byte1('Z') | ReadyForQuery is sent whenever the backend is ready for a new query cycle.  |
| Int32(5)   | Length of message contents in bytes, including self.  |
| Byte1      | Current backend transaction status indicator. Possible values are 'I' if idle (not in a transaction block); 'T' if in a transaction block; or 'E' if in a failed transaction block (queries will be rejected until block is ended). |

<a id="rowdescription"></a>
#### RowDescription 'T'

| Type       | Description |
|:-----------|:------------|
| Byte1('T') | Identifies the message as a row description. |
| Int32 | Length of message contents in bytes, including self. |
| Int16 | Specifies the number of fields in a row (may be zero). |
| Int32 | The number of elements in the type mapping pool, which is an array. This pool is used for non-native types like GEOMETRY and GEOGRAPHY (They are backed by LONG VARBINARY native type). |
| Then, for each type mapping element in the pool, there is the following: |  |
| Int32 | Specifies the base object ID of the non-native data type. |
| String | Specifies the name of the non-native data type. |
| Then, for each field in a row, there is the following: |  |
| String | The field name. |
| Int64 | If the field can be identified as a column of a specific table, the object ID of the table; otherwise zero. (denoted <strong><em>OID</em></strong> below). |
| String | The schema name. Only sent if <strong><em>OID</em></strong> != 0. |
| String | The table name. Only sent if <strong><em>OID</em></strong> != 0. |
| Int16 | If the field can be identified as a column of a specific table, the attribute number of the column; otherwise zero. The <strong><em>OID</em></strong> and the <strong><em>attribute number</em></strong> uniquely identify each field within the scope of a RowDescription message. |
| Int16 | If the column is a complex type, the <strong><em>attribute number</em></strong> of the field that is considered to be the parent; otherwise zero.<br>For example, an <em>ARRAY[INT]</em> will have one attribute number representing the ARRAY, and another attribute number representing the INT it contains. The INT will have a Parent that refers to the ARRAY.<br><em>New in version 3.10</em>: Before version 3.10, it only returned fields that did not have a parent. |
| Byte1 | '1' if the column uses the type mapping pool; '0' otherwise. |
| Int32 | If the column uses the type mapping pool, specifies the position (0-indexed) in the type mapping pool. Otherwise, specifies the object ID of the column's data type. |
| Int16 | The data type size. |
| Int16 | 1 if the column can contain null values; 0 otherwise. |
| Int16 | 1 if the column is identity; 0 otherwise. |
| Int32 | The type modifier. The meaning of the modifier is type-specific. |
| Int16 | The [format code](#formats-and-format-codes) being used for the field. |



#### VerifyFiles 'F'

| Type       | Description |
|:-----------|:------------|
| Byte1('F') | Identifies the message as a response to COPY FROM LOCAL command. The backend parses the file names out of the command, and sends them back to the frontend in this message. The frontend has to verify that these files exist and are readable before running the copy. |
| Int32      | Length of message contents in bytes, including self. |
| Int16      | The number of files (denoted ***N*** below). 0 if copy input is STDIN. |
| String\[***N***\] | The name of each data file to load.  |
| String     | The file name of Rejects.  |
| String     | The file name of Exceptions.  |

#### WriteFile 'O'

| Type       | Description |
|:-----------|:------------|
| Byte1('O') | Identifies the message as a response to COPY FROM LOCAL command.  |
| Int32      | Length of message contents in bytes, including self.  |
| String     | A file name if the command uses the REJECTED DATA and/or EXCEPTIONS parameters. Empty if the command uses the RETURNREJECTED parameters.  |
| Int32      | File length.  |
| String     | File content (not null-terminated).<br><br>Note: If the command uses the RETURNREJECTED parameter, file content (i.e. rejected row numbers) comes in **little-endian** Int64 format. <br>If the command doesn't use the RETURNREJECTED parameter and file content are very large (>8192 bytes), then file content are sent in chunks of 8192 bytes in multiple Writefile messages. |

The modified WriteFile format when 'extend_copy_reject_info' in the [StartupRequest](#startuprequest) message is set to *true* and the command uses the RETURNREJECTED parameter:

| Type       | Description |
|:-----------|:------------|
| Byte1('O') | Identifies the message as a response to COPY FROM LOCAL ... RETURNREJECTED command.  |
| Int32      | Length of message contents in bytes, including self.  |
| String     | File name. Empty because the command uses the RETURNREJECTED parameter.  |
| Int32      | File length.  |
| Then, for each rejected row: |  |
| Int64     | The rejected row number. **little-endian** format. |
| Int32     | Rejected message length.  **little-endian** format. |
| String    | Rejected message content (not null-terminated). |


## Error and Notice Message Fields

This section describes the fields that may appear in [ErrorResponse](#errorresponse-e) and [NoticeResponse](#noticeresponse-n) messages. Each field type has a single-byte identification token. Note that any given field type should appear at most once per message.

| Identification token | Description  |
|:---------------------|:------------|
| C                    | Code: the five-character SQLSTATE code for the error. See the "SQL state list" in Vertica Documentation for a list of all the SQLSTATE classes and values defined by Vertica. |
| D                    | Detail: additional information about the error message, in greater detail.  |
| F                    | File: the file name of the source-code location where the error was reported.  |
| H                    | Hint: an optional suggestion what to do about the problem. This is intended to differ from Detail in that it offers advice (potentially inappropriate) rather than hard facts. Might run to multiple lines. |
| L                    | Line: the line number of the source-code location where the error was reported.  |
| M                    | Message: the primary human-readable error message. Always present. |
| P                    | Position  |
| p                    | Internal position  |
| q                    | Internal query  |
| R                    | Routine: the name of the source-code routine reporting the error. |
| S                    | Severity: the field contents are ERROR, FATAL, or PANIC (in an error message), or WARNING, NOTICE, DEBUG, INFO, or LOG (in a notice message), or a localized translation of one of these. Always present.   |
| V                    | Error code: the numeric error code that Vertica reports. |
| W                    | Where: an indication of the context in which the error occurred. |


## Data Collector utility

Data Collector component ClientServerMessages (table `V_INTERNAL.dc_client_server_messages`) collects a subset of Client-Server Protocol Messages sent between the Front End and Back End. Not all protocol messages are currently logged by the Data Collector. Additionally, some messages has verbose data available.

#### Logged Frontend Messages (FE)

| Message (Character ID) | Contents logged  |
|:----------------------|:-----------------|
| Query ('Q') | Masked Query String |
| Parse ('P') |statement name: masked query string |
| Bind ('B')  | (empty)|
| Execute ('E') | portal name |
| Close ('C') | prepared statement/portal name |
| Describe ('D') | prepared statement/portal name |
| Flush ('H') |  (empty) |
| Sync ('S') |  (empty) |
| Terminate ('X') |  (empty) |
| Startup Request ('^') |  (empty) |
| Mars Request ('_') |  (empty) |
| SSL Request ('*') |  (empty) |
| Load Balance Request ('?') |  (empty) |
| Cancel Request ('!') | "Canceled Session" [canceled session ID] |
| Copy Data ('d') | (empty) |
| Copy Done ('c') | (empty) |
| Copy Error ('e') | (empty) |
| Copy Fail ('f') | (empty) |
| End of Batch Request ('j') | (empty) |


#### Logged Backend Messages (BE)

| Message (Character ID) | Contents logged  |
|:----------------------|:-----------------|
| Parse Complete ('1') | (empty) |
| Bind Complete ('2') | (empty) |
| Close Complete ('3') | (empty) |
| Portal Suspended ('s') | (empty) |
| Parameter Description ('t') | (empty) |
| No Data ('n') | (empty) |
| Load Balance Response('Y'/'N') |"load balance response" [yes/no] |
| Parameter Status ('S') |parameter name: parameter value |
| Command Complete ('C') |tag |
| Write File ('O') | (empty) |
| Load File ('H') | (empty) |
| Verify Files ('F') | (empty) |
| End of Batch Response ('J') | (empty) |
| Copy Done ('c') | (empty) |

### Setup and Usage

This DC Table is turned on by default. As with other session DC tables, this table contains columns for 'time', 'node_name', 'session_id', 'user_id', and 'user_name'. The unique columns for this DC table are 'source', 'message_type', and 'contents'. 

- The 'source' column describes whether the messages is a Frontend (FE) or Backend (BE) message.

- The 'message_type' column provides the character ID that distinguishes the type of message. Additionally, verbose messages will have a "+" appended to the single character message_type column. For example, the StartupRequest message KV pairs each are represented as a row in the table with the message_type "^+". 

- The 'contents' column lists the set of contents being logged, if any, for that message type.

#### Sample Query
```sql
=> SELECT source, message_type, contents FROM DC_CLIENT_SERVER_MESSAGES;
```

#### Clearing the DC Table
```sql
=> SELECT clear_data_collector('ClientServerMessages');
```

*Support since Server v12.0SP3*

## Server Debug Logging
 It is possible to enable the protocol debug log for the server. Execute the following SQL statement on the node to which the client connects:

```sql
=> SELECT set_debug_log('PROTOCOL','BASIC');
```

Now, the debug log messages in ***vertica.log*** can be filtered by the transaction ID.

Then, execute the following SQL statement to disable the protocol debug log after executing the client program:

```sql
=> SELECT clear_debug_log('PROTOCOL','BASIC');
```


## Summary of Changes since Protocol 3.0

### Protocol 3.16
- [OAuth 2.0 Authentication](#protocol-315) enhancement: Format change in the [AuthenticationOAuth](#authenticationoauth-r) message.

*Support since Server v24.1.0*

### Protocol 3.15
- [OAuth 2.0 Authentication](#protocol-315) enhancement: Format change in the [AuthenticationOAuth](#authenticationoauth-r) message to support OAuth browser workflow. 
- Workload support. Format change in the [StartupRequest](#startuprequest) message. New 'workload' parmeter allowing specification of workload name to be used by workload routing rules.
- Format change in [VerifiedFiles](#verifiedfiles-f) message.
- Format change in [WriteFile](#writefile-o) message when new 'protocol_features' parameter called 'extend-copy-reject-info' in [StartupRequest](#startuprequest) is set to true. In 3.15 only set to true by ODBC driver. This allows the ODBC driver users to properly get rejected row information for bulk loaded data through calls to SQLGetDiagRec.
- New [Frontend protocol messages](#logged-frontend-messages-fe) logged in DC_Client_Server_Messages table: [CopyData](#copydata-d), [CopyDone](#copydone-c), [CopyFail](#copyfail-f), [CopyError](#copyerror-e), [EndOfBatchRequest](#endofbatchrequest-j)
- New [Backend protocol messages](#logged-backend-messages-be) logged in DC_Client_Server_Messages table: [WriteFile](#writefile-o), [LoadFile](#loadfile-h), [VerifyFiles](#verifyfiles-f), [EndOfBatchResponse](#endofbatchresponse-j), [CopyDone](#copydone-c)


*Support since Server v23.3.0*

### Protocol 3.14

Changes include:
- client_os_hostname support. Format change in the [StartupRequest](#startuprequest) message. New 'client_os_hostname' parameter. Allow client's OS hostname to be recorded in Sessions Table.

*Support since Server v12.0SP4*

### Protocol 3.13

Changes include:
- Postgres compatibility support. Format change in the [StartupRequest](#startuprequest) message. New 'protocol_compat' parameter allowing specification of protocol understood by the driver.

*Support since Server v12.0SP3*

### Protocol 3.12

Changes include:

- Extended Complex Type support. Format change in the [StartupRequest](#startuprequest) message. New 'protocol_features' parameter supporting 'request_complex_types'.
- Fall-Through Authentication Filtering: Format change in the [StartupRequest](#startuprequest) message. New 'auth_category' parameter.
- [OAuth 2.0 Authentication](#protocol-312) enhancement: add [AuthenticationOAuth](#authenticationoauth-r) message.

*Support since Server v12.0SP2*

### Protocol 3.11

Changes include:

- [OAuth 2.0 Authentication](#oauth2-authentication) support: Format change in the
[StartupRequest](#startuprequest)
message. New 'oauth_access_token' parameter.

*Support since Server v11.1SP1*

### Protocol 3.10

Changes include:

- Complex Types support (JDBC only): Format change in the [RowDescription](#rowdescription-t) message. OID change for 1D Arrays & Sets.

*Support since Server v11.0*

### Protocol 3.9
Changes include:

JDBC Complex Type Metadata support: Server changed the way it represents some column metadata in v_catalog.types table.

*Support since Server v10.1SP1*

### Protocol 3.8

Changes include:

- Native UUID type support: Present the column as type UUID (object ID = 20) instead of type CHAR (object ID = 8).

*Support since Server v9.0*

### Protocol 3.7

Changes include:

- Client Backward compatibility (Server's protocol version is older than the client's): Format change in the [StartupRequest](#startuprequest)
message.
  - The existing protocol_version parameter is now sent back to the client in a [ParameterStatus](#parameterstatus-s) message. It contains the protocol version supported by the server.

*Support since Server v8.0*

<br>

> The following has been verified by reviewing the startup method in the JDBC driver's protocol stream in the 9.1 source branch. The exact protocol version each was added is TBD.

### Protocol 3.6

Changes include:

- Format change in the [StartupRequest](#startuprequest) message. New parameters added:
  - mars
  - client_os_user_name
- New MARS related messages.

*Support since Server v7.2*

### Protocol 3.5

Changes include:

- SHA512 authentication support:
  - New [AuthenticationHashSHA512Password](#authenticationhashsha512password-r)
message
  - New [AuthenticationHashPassword](#authenticationhashpassword-r)
message

- Format change in the [StartupRequest](#startuprequest) message. New parameters added:
  - protocol_version
  - binary_data_protocol

- Server backward compatibility (Client's protocol version is older than the server's)

*Support since Server v7.1*

### Protocol 3.4 and before

These are functional but buggy versions. Changes include:
- Server returns the error code (V) in [ErrorResponse](#errorresponse-e) and [NoticeResponse](#noticeresponse-n) messages.
- [Connection Load Balancing](#connection-load-balancing) support.
- User-defined Types support: Format change in the [RowDescription](#rowdescription-t) message.
- Support only one portal. The portal name required in some messages is actually ignored by the backend.
- New [CommandDescription](#commanddescription-m) message.

> Note: As of protocol version 3.4, the [StartupRequest](#startuprequest) message includes the following parameters:
> - user
> - database
> - client_pid
> - client_label
> - client_type
> - client_version 
> - client_os

*Support protocol 3.4 since Server v7.0*