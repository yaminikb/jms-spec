# More flexible JMS MDBs (Updates to version 2)</h1>

This page contains some updates to  [[JMSListener2|version 2 of the proposals]]  to simplify the configuration of JMS MDBs in JMS 2.1 and Java EE 8. 

Comments are invited. See [https://java.net/projects/jms-spec/pages/JMS21#How_to_get_involved_in_JMS_2.1 How to get involved in JMS 2.1].

This page will be extended with additional changes and points for discussion. When a reasonable degree of agreement has been received, an updated version (version 3) of the proposals will be added.

__TOC__

==Changes from version 2==

The major issues which still need to be decided are:

* Issue I17: Should multiple callback methods be permitted? As issue I18 describes, there is an argument that allowing multiple callback methods may be confusing for developers, who may not realise the concurrency implications (i.e. that defining multiple callbacks reduces the number of MDB instances available to process each callback unless the MDB poolsize is increased.). It may also make implementation more complex for application servers that automatically calculate the size of the MDB pool. It may also require an excessive amount of extra work for vendors which offer monitoring and management features for JMS MDBs, since they might need to be extended to allow each callback to be managed separately. Finally, it introduces an ambiguity as to how old-style activation properties relate to multiple callback methods. For example, what is the effect of setting the activation property clientId? when there are multiple callback methods, each using a separate connection?  

* Whether we should allow the new annotations to be combined with old-style activation properties. The current proposals state that activation properties may be used to override new-style annotations. However this introduces additional test scenarios. It also introduces potential ambiguity if there are multiple callback methods. However these are still MDBs, and activation properties are an intrinsic feature of both MDBs and the JCA API for message endpoints. One possible clarification is to state that activation properties *defined in the EJB spec* will override those implied by new-style annotations, and that the effect of setting any non-standard activation properties (e.g. other ways to specify the destination) is not defined by the spec - just like the way that the current spec does not define how spec-defined annotations interact with non-standard annotations.

===Relationship between the old and new ways to define a JMS MDB===

Should these new annotations be allowed for MDBs that implement the <tt>javax.jms.MessageListener</tt> interface? 

There are two possible cases we need to consider:

* Allowing these new annotations to be used on the legacy <tt>onMessage</tt> method of a <tt>javax.jms.MessageListener</tt><p/>[[JMSListener2#Specifying_the_callback_method|Version 2]]  proposed any of the new annotations could be specified. However the requirement that the <tt>@JMSListener</tt> always be specified could not apply since that would break existing MDBs.

* Allowing these new annotations to be used to define additional callback methods (i.e. in addition to the <tt>onMessage</tt> method).<p/>[[JMSListener2#Specifying_the_callback_method|Version 2]]  proposed that this be allowed so long as the MDB also implemented the <tt>javax.jms.JMSMessageDrivenBean</tt> marker interface. 

On reflection, this is probably introducing unnecessary complexity. It means that lots of additional use cases need to be defined in the spec, implemented, and tested, whilst bringing little benefits to users who can convert to a new-style JMS MDB easily enough.

In addition, since we're seeking EJB spec changes to remove the need for the <tt>javax.jms.JMSMessageDrivenBean</tt> marker interface completely, it doesn't make any sense to introduce a requirement to use it here.

It is therefore proposed to make a clear distinction between "legacy" JMS MDBs and "new-style" JMS 2.1 MDBs:

* Legacy JMS MDBs will implement <tt>javax.jms.MessageListener</tt> and will be configured as they are now. 

* New-style MDBs will not implement <tt>javax.jms.MessageListener</tt>.  They will use the new annotations to specify the callback methods. They will also implement the <tt>javax.jms.JMSMessageDrivenBean</tt> marker interface, though we're seeking EJB spec changes to remove the need for this.

* If the new annotations are used on a MDB which implements <tt>javax.jms.MessageListener</tt> then deployment must fail.

===New JMSListenerProperty annotation===

We need to define an additional annotation to allow proprietary activation properties to be specified on the callback method. Many application servers (and resource adapters) use these to offer additional non-standard features. Examples of such properties are the Glassfish-specific activation properties <tt>reconnectAttempts</tt> and <tt>reconnectInterval</tt>, though just about every other application server or resource adapter defines its own set of proprietary activation properties.

Without such an annotation, applications would have to continue defining these properties in the same way as now, thereby forcing applications to mix the new JMS-specific method annotations on the callback method with the old generic class annotations:
<p/><p/>
 @MessageDriven(activationConfig = {
   @ActivationConfigProperty(propertyName = "foo1", propertyValue = "bar1"),
   @ActivationConfigProperty(propertyName = "foo2", propertyValue = "bar2")
 })
 public class MyMessageBean implements JMSMessageDrivenBean {
 
  @JMSListener(lookup="java:global/Trades", type=JMSListener.Type.QUEUE)
   public void processTrade(TextMessage tradeMessage){
     ...
   }
 
 }
<p/>
It is therefore proposed that a new method annotation <tt>@JMSListenerProperty</tt> be defined which the application can use to specify arbitrary activation properties. This annotation would be a "repeatable annotation" so that it could be used multiple times to set multiple properties.
<p/><p/>
 @MessageDriven
 public class MyMessageBean implements JMSMessageDrivenBean {
 
  <b>@JMSListenerProperty(name="foo1", value="bar2")</b>
  <b>@JMSListenerProperty(name="foo2", value="bar2")</b>
  @JMSListener(lookup="java:global/Trades", type=JMSListener.Type.QUEUE)
   public void processTrade(TextMessage tradeMessage){
     ...
   }
 
 }
<p/>
Since this annotation is a repeatable annotation, a composite annotation needs to be defined as well which the compiler will insert automatically when it encounters a repeatable annotation. This will be called <tt>@JMSListenerProperties</tt>.  


{|- border="1"
! New or modified?
! Interface or annotation?
! Name
| Link to javadocs
|-
| New
| Method annotation
| <tt>javax.jms.JMSListenerProperty</tt>
| [https://jms-spec.java.net/2.1-SNAPSHOT/apidocs/javax/jms/JMSListenerProperty.html javadocs]
|-
| New
| Method annotation
| <tt>javax.jms.JMSListenerProperties</tt>
| [https://jms-spec.java.net/2.1-SNAPSHOT/apidocs/javax/jms/JMSListenerProperties.html javadocs]
|} 


Since this gives us yet another way to define activation properties we need to define some override rules:

* A property set using the existing activation property annotations or XML elements will override any value set using the <tt>@JMSListenerProperty</tt> annotation. This allows values to be overridden (for the MDB as a whole) by defining activation properties in the deployment descriptor.

* If <tt>@JMSListenerProperty</tt>  is used to specify one of the JMS standard activation properties then this will override any value set using the corresponding new JMS-specific annotation. This order is chosen to be consistent with the previous rule.

===Callback methods that throw exceptions===

Version 2 [[JMSListener2#Specifying_the_callback_method|proposed]] that callback methods will be allowed to throw exceptions as follows:

<table style="margin-left:16px"> <tr> <td>
* Callback methods will be allowed to declare and throw exceptions. Checked exceptions and <tt>RuntimeException</tt> thrown by the callback method (including those thrown by the <tt>onMessage</tt> method of a <tt>MessageListener</tt>) will be handled by the EJB container as defined in the EJB 3.2 specification section 9.3.4 "Exceptions thrown from Message-Driven Bean Message Listener methods".  This defines whether or not any transaction in progress is committed or rolled back, depending on whether or not the exception is a "system exception" or an "application exception", whether or not the application exception is specified as causing rollback, and whether or not the application has called <tt>setRollbackOnly</tt>. It also defines whether or not the MDB instance is discarded. If the transaction is rolled back, or a transaction is not being used, then the message will be redelivered. 

* The JMS provider should detect repeated attempts to redeliver the same message to a MDB. Such messages may either be discarded or delivered to a provider-specific dead message queue. (Note that this not completely new to JMS: JMS 2.1 section 8.7 refers to a JMS provider "giving up" after a message has been redelivered a certain number of times).
</td></tr></table>

Version 3 expands on this by reviewing how exceptions thrown by old-style MDBs should be handled now, and uses this as the basis for proposals on how exceptions thrown by new-style MDBs should be handled.

====A review of how <tt>RuntimeException</tt>s thrown by old-style MDBs are handled====

With old-style JMS MDBs (those that implement <tt>javax.jms.MessageListener</tt>) the <tt>onMessage</tt> callback method is prevented by the compiler from declaring or throwing checked exceptions. However the compiler does allow them to throw unchecked exceptions and the existing EJB and JMS specifications do define how <tt>RuntimeException</tt>s should be handled.

(Recap: an ''unchecked exception'' is a class which extends <tt>RuntimeException</tt> or <tt>Error</tt>. A ''checked exception'' is any other class which extends <tt>Throwable</tt>. The existing JMS and EJB specifications don't define the behaviour for <tt>Error</tt> exceptions such as <tt>ThreadDeath</tt> and <tt>NoClassDefFoundError</tt>, no doubt because these are by definition "serious problems that a reasonable application should not try to catch". This is probably why the specifications refer to <tt>RuntimeException</tt>s rather than "unchecked exceptions"; JMS 2.1 will take the same approach.)

In deciding how old-style MDBs should handle <tt>RuntimeException</tt>s there are several places we need to look:

* JMS 2.0 section 8.7 "Receiving messages asynchronously" (reproduced below) defines how a JMS provider should handle a <tt>RuntimeException</tt> thrown by the <tt>onMessage</tt> method  of a <tt>MessageListener</tt>. However this section only considers Java SE acknowledgement modes. That means that it is not relevant to MDBs which consume messages in a container-managed transaction. However it ''is'' relevant for MDBs which consume messages in auto-acknowledge or dups-ok-acknowledge modes - which is the case when bean-managed transactions are specified. It says that in this case the message will be "immediately redelivered", where "the number of times a JMS provider will redeliver the same message before giving up is provider-dependent".

<table style="margin-left:16px"> <tr> <td>
8.7 Receiving messages asynchronously
<p/>
A client can register an object that implements the JMS MessageListener interface with a consumer. As messages arrive for the consumer, the provider delivers them by calling the listener’s onMessage method.
<p/>
It is possible for a listener to throw a RuntimeException; however, this is considered a client programming error. Well behaved listeners should catch such exceptions and attempt to divert messages causing them to some form of application-specific ‘unprocessable message’ destination.
<p/>
The result of a listener throwing a RuntimeException depends on the session’s acknowledgment mode.
<p/>
<ul>
<li>AUTO_ACKNOWLEDGE or DUPS_OK_ACKNOWLEDGE - the message will be immediately redelivered. The number of times a JMS provider will redeliver the same message before giving up is provider-dependent. The JMSRedelivered message header field will be set, and the JMSXDeliveryCount message property incremented, for a message redelivered under these circumstances.</li>
<li>CLIENT_ACKNOWLEDGE - the next message for the listener is delivered. If a client wishes to have the previous unacknowledged message redelivered, it must manually recover the session.</li>
<li>Transacted Session - the next message for the listener is delivered. The client can either commit or roll back the session (in other words, a RuntimeException does not automatically rollback the session).</li>
</ul>
<p/>
JMS providers should flag clients with message listeners that are throwing RuntimeException as possibly malfunctioning.
<p/>
See Section 6.2.13 “Serial execution of client code” for information about how onMessage calls are serialized by a session.
</td></tr></table>

* EJB 3.2 specification section 9.3.4 "Exceptions thrown from Message-Driven Bean Message Listener methods" describes how the EJB container should handle exceptions thrown by the MDB's callback method. It specifies whether any transaction is committed or rolled back, whether the MDB is discarded, and what exception is rethrown to the resource adapter (or JMS provider). However it does not define how the resource adapter (or JMS provider) should handle any such exception. For that we need to look at the JMS specification.

* In the case where a container-managed transaction is being used, the EJB specification defines whether or not the transaction will be committed by the container or rolled back. If the transaction is committed then it is clear that the message being received will be considered to have been successfully delivered and will not be delivered again. If the transaction is rolled back then the message is clearly eligible for redelivery. JMS 2.0 section 6.2.7 "Transactions" (which mentions Java EE transactions as well as Java SE local transactions) states that  "if a transaction rollback is done, its produced messages are destroyed and its consumed messages are automatically recovered."
<p/>
Considering the EJB 3.2 specification and JMS 2.0 specifications together, here is a summary the existing requirements for handling a <tt>RuntimeException</tt> thrown by the <tt>onMessage</tt> method of a JMS MDB:
<p/>
<table border="1">
<tr><th colspan="4">Existing rules for handling a <tt>RuntimeException</tt> thrown by old-style <tt>onMessage</tt></th></tr>
<tr><th>Transactional mode</th><th>Type of RuntimeException</th><th>Container's action<br/> (as defined by EJB 3.2 section 9.3.4)</th><th>Resource adapter's action (as defined by JMS 2.0 specification)</th></tr>

<tr>
<td rowspan="4">Container-managed transaction demarcation configured (which is the default) and callback method configured with <tt>Required</tt> attribute (which is also the default)<br/><br/>  (Message is received in a transaction, and callback method is called in the same transaction)</td>

<td><tt>RuntimeException</tt> is annotated with <tt>@ApplicationException(rollback="true")</tt></td>
<td>Rollback transaction.<br/>Rethrow exception to resource adapter.</td>
<td>Message is "automatically recovered" (JMS 2.0 section 6.2.7 applies)</td>

</tr>

<tr>
<td><tt>RuntimeException</tt> is annotated with <tt>@ApplicationException(rollback="false")<br/> (default) and MDB calls <tt>setRollbackOnly</tt></tt></td>
<td>Rollback transaction.<br/>Rethrow exception to resource adapter.</td>
<td>Message is "automatically recovered" (JMS 2.0 section 6.2.7 applies)</td>
</tr>

<tr>
<td><tt>RuntimeException</tt> is annotated with <tt>@ApplicationException(rollback="false")</tt><br/>(default) and MDB does not call <tt>setRollbackOnly</tt></td>
<td>Commit transaction <br/>Rethrow exception to resource adapter.</td>
<td>Continue with next message.</td>
</tr>

<tr>
<td>Any other <tt>RuntimeException</tt></td><td>Rollback transaction.<br/>
Discard MDB instance<br/>
Wrap exception in <tt>EJBException</tt> and rethrow to resource adapter.</td><td>Message is "automatically recovered" (JMS 2.0 section 6.2.7 applies)</td>
</tr>

<tr><td rowspan="2">Container-managed transaction demarcation configured (which is the default) and callback method configured with <tt>NotSupported</tt> attribute <br/><br/>(This case is not well defined in EJB 3.2, but this is assumed to mean that no transaction is used either to receive the message or to execute the callback method. Instead  the message will be received in auto-acknowledge or dups-ok-acknowledge mode.)</td><td><tt>RuntimeException</tt> is annotated with <tt>@ApplicationException</tt></td><td><tt>Rethrow exception to resource adapter.</tt></td><td>Redeliver message immediately. May "give up" after repeated attempts. (JMS 2.0 section 8.7 applies)</td></tr>
<tr><td>Any other <tt>RuntimeException</tt></td><td>Discard MDB instance<br/>
Wrap exception in <tt>EJBException</tt> and rethrow to resource adapter.</td><td>Redeliver message immediately. May "give up" after repeated attempts. (JMS 2.0 section 8.7 applies)</td></tr>

<tr>
<td rowspan="2">Bean-managed transaction<br/><br/>(Message is received in auto-ack or dups-ok-ack mode, not in a transaction. However <tt>onMessage</tt> may itself start and commit a transaction.)</td><td><tt>RuntimeException</tt> is annotated with <tt>@ApplicationException</tt></td><td>Rethrow exception to resource adapter.</td><td>Redeliver message immediately. May "give up" after repeated attempts. (JMS 2.0 section 8.7 applies)</td>
</tr>

<tr><td>Any other <tt>RuntimeException</tt></td><td>Rollback transaction if there is one in progress (this does not affect the message being delivered)<br/>
Discard MDB instance<br/>
Wrap exception in <tt>EJBException</tt> and rethrow to resource adapter.</td><td>Redeliver message immediately. May "give up" after repeated attempts. (JMS 2.0 section 8.7 applies)</td></tr>

</table>

(EJB 3.2 defines that the behaviour of the container depends on whether the exception is an "application exception" or a "system exception".  If the exception is a <tt>RuntimeException</tt> then it is assumed to be a system exception unless it is explicitly annotated as an application exception.)

These existing rules leave scope for clarification:

* JMS 2.0 section 8.7 really applies only to the auto-acknowledge and dups-ok-acknowledge cases when using a Java SE message listener. The specification does not explicitly state that it applies also to the auto-acknowledge and dups-ok-acknowledge cases when using a MDB. 

* The statement that the JMS provider may "give up" after repeated attempts to deliver a message applies only to the auto-acknowledge and dups-ok-acknowledge cases. The specification does not explicitly state whether the same option applies to a message that is repeatedly rolled back.

* The term "give up" is not defined. The specification does not state whether this means the message is permanently deleted or whether it might be redelivered at some future time.  It also does not mention the possibility of diverting the message to a dead message queue.

====Discussion of how exceptions thrown by new-style MDBs should be handled====

In a new-style JMS MDB,  it will be possible for a callback method to throw a <tt>RuntimeException</tt>. This is identical to the existing case of an old-style MDB whose <tt>onMessage</tt> method throws a  <tt>RuntimeException</tt>, and the existing rules, summarised in the previous section, can apply. We need to decide whether to state that throwing a <tt>RuntimeException</tt> is considered a programming error.

In addition, it is proposed that callback methods on new-style MDBs be allowed to declare and throw checked exceptions. It will be up to the MDB provider to decide what checked exceptions may be thrown; this will ''not'' be considered a programming error. 

The EJB 3.2 specification already defines how the EJB container should handle checked exceptions thrown by a MDB's callback method (although this has never previously been possible with JMS MDBs). Here's a summary of what the existing EJB and JMS specifications define when the callback method of a new-style MDB throws a checked exception.:

<p/>
<table border="1">
<tr><th colspan="4">Existing rules for handling a checked exception thrown by callback method in new-style MDB</th></tr>
<tr><th>Transactional mode</th><th>Type of checked exception</th><th>Container's action<br/> (as defined by EJB 3.2 section 9.3.4)</th><th>Resource adapter's action (as defined by JMS 2.0 specification)</th></tr>
<tr><td rowspan="3">Container-managed transaction demarcation configured (which is the default) and callback method configured with <tt>Required</tt> attribute (which is also the default)<br/><br/>  (Message is received in a transaction, and callback method is called in the same transaction)</td>
<td>Checked exception is annotated with <tt>@ApplicationException(rollback="true")</tt></td>
<td>Rollback transaction.<br/>Rethrow exception to resource adapter.</td>
<td>Message is "automatically recovered" (JMS 2.0 section 6.2.7 applies)</td>
</tr>
<tr><td>Any other checked exception and <br/>MDB called <tt>setRollbackOnly</tt>
</td>
<td>Rollback transaction.<br/>Rethrow exception to resource adapter.</td>
<td>Message is "automatically recovered" (JMS 2.0 section 6.2.7 applies)</td>
</tr>

<tr><td>Any other checked exception and <br/>MDB did not call <tt>setRollbackOnly</tt></td>
<td>Commit transaction.<br/>Rethrow exception to resource adapter.</td>
<td>Transaction was committed so continue with next message</td>
</tr>

<tr>
<td rowspan="1">Container-managed transaction demarcation configured (which is the default) and callback method configured with <tt>NotSupported</tt> attribute <br/><br/>(This case is not well defined in EJB 3.2, but this is assumed to mean that no transaction is used either to receive the message or to execute the callback method. Instead  the message will be received in auto-acknowledge or dups-ok-acknowledge mode.)</td><td>Any checked exception</td><td><tt>Rethrow exception to resource adapter.</tt></td>
<td>Not defined in JMS 2.0</td>
</tr>

<tr>
<td rowspan="1">Bean-managed transaction<br/><br/>(The message is received in auto-ack or dups-ok-ack mode, not in a transaction. However <tt>onMessage</tt> may itself start and commit a transaction.)</td><td>Any checked exception</td><td>Rethrow checked exception to resource adapter.</td>
<td>Not defined in JMS 2.0</td>
</tr>
</table>
<p/>
(In EJB 3.2 parlance, any checked exception is considered an "application exception", irrespective of whether it is explicitly annotated with <tt>@ApplicationException</tt>. By default an application exception does not prevent the transaction being committed. The annotation <tt>@ApplicationException(rollback="true")</tt> may be used to specify transaction rollback.)

As the table above shows, the existing specifications already cover the case where the message is being received in a container-managed-transaction, but not the cases where the message is being received in auto-ack or dups-ok-ack mode. So what do we need to do for JMS 2.1? 

The minimum we need to do for JMS 2.1 is:

* Say nothing about the case where the message is being received in a container-managed-transaction and the callback method throws a checked exception. The EJB 3.2 specification already defines when the transaction is rolled back and when it is committed. Automatic recovery after a rollback is not a new case so doesn't need to be defined further. Note that this would mean there is no mention of the case where the same message repeatedly causes an exception to be thrown. 

* Define the required behaviour for when the message is being received in auto-ack or dups-ok-ack mode and the callback method throws a checked exception. We could simply define that the result is the same as when the callback throws a <tt>RuntimeException</tt>, which is already defined for the Java SE case in JMS 2.0 section 8.7. This would say that the message would be immediately redelivered, that the redelivered flag should be set, and that the JMS provider may "give up" if a message is repeatedly redelivered. For completeness we could also extend this to cover a <tt>RuntimeException</tt> thrown by a MDB.

See [ https://java.net/projects/jms-spec/pages/JMSListener3#Proposed_minimum_new_wording_for_JMS_2.1_specification Proposed minimum new wording for JMS 2.1 specification] below.

However we may decide that we want to do more than the minimum:

* We could say more about how a message is "automatically recovered" (which are existing words) after a Java EE transaction rollback. We could mention that recovery might not be immediate and so might occur out of order (though since the spec doesn't describe MDB message order this may not be necessary). We could mention whether or not the redelivered flag should be set on a recovered message. We could mention what the resource adapter (if used) or JMS provider is allowed to do if the same message is repeatedly recovered. 

* We could say more about how a message is "immediately redelivered" after an exception is thrown when auto-ack or dups-ok-ack is being used. The wording in JMS 2.0 8.7 doesn't allow for the possibility that there may be more than one consumer on the queue (or JMS 2.0 shared subscription) , or that there may be more than one MDB instance.

* We could say more about what it means to "give up" on a message which is being repeatedly redelivered/recovered. Does this mean deleting the message? Should we mention the possibility of diverting the message to a dead message queue?

See [https://java.net/projects/jms-spec/pages/JMSListener3#Proposed_extended_new_wording_for_JMS_2.1_specification Proposed extended new wording for JMS 2.1 specification] below.

====Proposed minimum new wording for JMS 2.1 specification====

Here is a proposed minimum wording. This would be a new section (arbitrarily numbered 16.5 here) in a new chapter 16 defining JMS MDBs in more detail. It will follow a number of previous sections which define how JMS MDBs are configured.

<table style="margin-left:16px"> <tr> <td>
<h3>16. JMS message-driven beans</h3>
<h4>16.5 Exceptions thrown by message callback methods</h4>
<p/>
An application-defined callback method of a JMS MDB may throw checked exceptions (where allowed by the method signature) or <tt>RuntimeException</tt>s. 
<p/>
The <tt>onMessage</tt> method of a JMS MDB that implements <tt>MessageListener</tt> may throw <tt>RuntimeException</tt>s.
<p/>
All exceptions thrown by message callback methods must be handled by the container as defined in the EJB 3.2 specification section 9.3.4 "Exceptions thrown from Message-Driven Bean Message Listener methods". This defines whether or not any container-managed transaction is committed or rolled back by the container. It also defines whether or not the MDB instance is discarded, whether or not the exception is required to be logged, and what exception is re-thrown to the resource adapter (if a resource adapter is being used).
<p/>
If a resource adapter is being used it must catch any checked exceptions or <tt>RuntimeException</tt>s thrown by the callback method. 
<p/>
If a message is being delivered to the callback method of a MDB using container-managed transaction demarcation, and the resource adapter had called the <tt>beforeDelivery</tt> method on the <tt>javax.resource.spi.endpoint.MessageEndpoint</tt> prior to invoking the callback method then it must call the <tt>afterDelivery</tt> method afterwards even if the callback method threw a checked exception or <tt>RuntimeException</tt>. This ensures that the container-managed transaction is rolled back or committed by the container as required by the EJB specification.
<p/>
If a message is being delivered to the callback method of a MDB, and auto-acknowledge or dups-ok-acknowledge mode is being used, and the callback method throws a checked exception or a <tt>RuntimeException</tt> then the message will be automatically redelivered.  The number of times a JMS provider will redeliver the same message before giving up is provider-dependent. The JMSRedelivered message header field will be set, and the JMSXDeliveryCount message property incremented, for a message redelivered under these circumstances.

</td></tr></table>
<p/>

The proposed minimum wording above essentially extends the wording already used in the JMS specification for Java SE message listeners to cover JMS MDBs:

* There's a reminder that EJB 3.2 defines whether or not any container-managed transaction is committed or rolled back, and a requirement that the appropriate <tt>afterDelivery</tt> method must be called even if an exception is thrown. 

* If auto-acknowledge or dups-ok-acknowledge mode is being used then the behaviour of the resource adapter or JMS provider is similar to that defined in JMS 2.0 section 8.7.

* The proposed wording above not repeat the statement in JMS 2.0 section 8.7 that "it is possible for a listener to throw a <tt>RuntimeException</tt>; however, this is considered a client programming error.". That statement remains for Java SE message listeners for reasons of backward compatibility. However we're not repeating it here for MDBs since we're explicitly allowing MDB callback methods to throw checked exceptions, and it seems arbitrary to allow checked exceptions but to discourage <tt>RuntimeException</tt>s. Especially as methods in the JMS 2.0 simplified API throw <tt>RuntimeException</tt>s.

====Proposed extended new wording for JMS 2.1 specification====

Here is a proposed extended wording. 

The JMS 2.1 specification will contain a new chapter 16 defining JMS MDBs in more detail. This chapter will have a section (arbitrarily numbered 16.5 here) devoted to exceptions thrown by callback methods and a section following it (arbitrarily numbered 16.6 here)  about message redelivery in general. Here is a suggested wording. This section will follow a number of previous sections which define how JMS MDBs are configured.

<table style="margin-left:16px"> <tr> <td>
<h3>16. JMS message-driven beans</h3>
<h4>16.5 Exceptions thrown by message callback methods</h4>
<p/>
An application-defined callback method of a JMS MDB may throw checked exceptions (where allowed by the method signature) or <tt>RuntimeException</tt>s. 
<p/>
The <tt>onMessage</tt> method of a JMS MDB that implements <tt>MessageListener</tt> may throw <tt>RuntimeException</tt>s.
<p/>
All exceptions thrown by message callback methods must be handled by the container as defined in the EJB 3.2 specification section 9.3.4 "Exceptions thrown from Message-Driven Bean Message Listener methods". This defines whether or not any container-managed transaction is committed or rolled back by the container. It also defines whether or not the MDB instance is discarded, whether or not the exception is required to be logged, and what exception is re-thrown to the resource adapter (if a resource adapter is being used).
<p/>
If a resource adapter is being used it must catch any checked exceptions or <tt>RuntimeException</tt>s thrown by the callback method. 
<p/>
If a message is being delivered to the callback method of a MDB using container-managed transaction demarcation, and the resource adapter had called the <tt>beforeDelivery</tt> method on the <tt>javax.resource.spi.endpoint.MessageEndpoint</tt> prior to invoking the callback method then it must call the <tt>afterDelivery</tt> method afterwards even if the callback method threw a checked exception or <tt>RuntimeException</tt>. This ensures that the container-managed transaction is rolled back or committed by the container as required by the EJB specification.
<p/>
If a message is being delivered to the callback method of a MDB using container-managed transaction demarcation, and the transaction is rolled back, then the message will be automatically recovered. Message delivery after transaction recovery is described in more detail in section 16.6.1 "Message redelivery after transaction rollback".
<p/>
If a message is being delivered to the callback method of a MDB, and auto-acknowledge or dups-ok-acknowledge mode is being used, and the callback method throws a checked exception or a <tt>RuntimeException</tt>, then the message will be automatically redelivered.  The message will not be acknowledged and will be immediately redelivered. Message redelivery after an exception in auto-acknowledge or dups-ok-acknowledge mode is described in more detail in section 16.6.2 "Message redelivery after an exception, in auto-acknowledge or dups-ok-acknowledge mode".

<h4>16.6 Message redelivery to MDBs</h4>

<h5>16.6.1 Message redelivery after transaction rollback</h5>
If a message is being delivered to the callback method of a MDB, and container-managed transaction demarcation is being used, then if the transaction is rolled back for any reason the message will be redelivered. Redelivery may not be immediate. The number of times a JMS provider will redeliver the same message before giving up is provider-dependent. The JMSRedelivered message header field will be set, and the JMSXDeliveryCount message property incremented, for a message redelivered under these circumstances. 

<h5>16.6.2 Message redelivery after an exception, in auto-acknowledge or dups-ok-acknowledge mode</h5>
If a message is being delivered to the callback method of a MDB, and auto-acknowledge or dups-ok-acknowledge mode is being used, and the callback method throws a checked exception or a RuntimeException then the message will be immediately redelivered. The number of times a JMS provider will redeliver the same message before giving up is provider-dependent. The JMSRedelivered message header field will be set, and the JMSXDeliveryCount message property incremented, for a message redelivered under these circumstances. 

</td></tr></table>
<p/>
The proposed extended wording above extends the minimum wording as follows:

* There's a new section which defines how messages are redelivered after rollback. This states that redelivery may not be immediate. It also states that the redelivery flag should be set , that the redelivery count should be incremented and that the resource adapter may "give up" after repeated redelivery.

====Redelivery delays, redelivery limits and dead-letter queues====

The proposals above could be extended to include a fuller set of features which allow the user to configure

* A minimum time delay between redelivery attempts (which will inevitably change message order)

* The maximum number of times a message can be redelivered before some other action is taken

* The action to be taken when the redelivery limit has been reached. Options could be
** delete the message
** forward the message to a specified queue or topic

The proposals above will define the behaviour when none of these are specified. 

Adding these will be considered separately. (See also [https://java.net/jira/browse/JMS_SPEC-117 JMS_SPEC-117]).

==Simplifying the method annotations: some options==

No changes are proposed to the method annotations (apart from the addition of <tt>JMSListenerProperty</tt>) at this stage.  

(The term "method annotations" here refers to the annotations that are placed on each callback method rather than the annotations that are placed on method arguments)

However the current proposals still need to be reviewed in detail. Aspects needing review include
* Exactly what annotations we require, and what attributes each annotation has
* Exactly what name should be used for each annotation (e.g. should they start with "JMS"), and what attributes they should have
* Whether attributes are optional (e.g. have a default) or mandatory
* The enumerated types used

===Recap of current proposal (Option A) ===

The following example illustrates the annotations currently proposed:
<p/>
 @JMSListener(lookup="java:global/java:global/Trades",type=JMSListener.Type.TOPIC )
 @JMSConnectionFactory("java:global/MyCF")
 @SubscriptionDurability(SubscriptionDurability.Mode.DURABLE)
 @ClientId("myClientID1")
 @SubscriptionName("mySubName1")
 @MessageSelector("ticker='ORCL'")
 @JMSListenerProperty(name="foo1", value="bar1")
 @JMSListenerProperty(name="foo2", value="bar2")
 @Acknowledge(Acknowledge.Mode.DUPS_OK_ACKNOWLEDGE)
 public void giveMeAMessage(Message message) {
   ...
 }
<p/>
Obviously, this is an extreme example which demonstrates the use of every annotation. 

The most important annotation here is <tt>@JMSListener</tt>. This has two attributes, "<tt>lookup</tt>" and "<tt>type</tt>"

* <tt>@JMSListener</tt> is the annotation that signifies that this is a callback method, and is always required. That's why the name of this annotation is <tt>@JMSListener</tt> rather than, say,  <tt>@JMSDestination</tt>, despite its role being to specify the queue or topic.

* The <tt>lookup</tt> attribute is used to set the <tt>destinationLookup</tt> activation property and can be omitted. This allows the MDB to use legacy or non-standard ways to define the queue or topic, such as  using a non-standard activation property (specified using a <tt>@JMSListenerProperty</tt> annotation). 

* The "<tt>type</tt>" attribute is used to set the <tt>destinationType</tt> activation property, which specifies whether the destination is a queue or topic. This is proposed as a mandatory attribute which must be set (or the code won't compile). See the side discussion below.

* We may want to extend <tt>@JMSListener</tt> in the future to allow the queue or topic's "non-portable" name to be specified. That's not proposed currently (it's a separate issue) but we did we could define an additional attribute "<tt>name</tt>", as in <tt>@JMSListener(name="tradesTopic",type=JMSListener.Type.TOPIC )</tt>. That's why the attribute used to specify JNDI name should be called "<tt>lookup</tt>" rather than "<tt>value</tt>".

The remaining annotations have a single attribute "<tt>value</tt>", with each annotation corresponding to a single activation property.

There are a number of things we should review here. Probably the main thing we need to consider is which is better:

1. to have lots of separate annotations, almost all of which have a single attribute, or

2. to have a smaller number of annotations, each of which set multiple attributes.

<p/>
<table> <tr style="background-color:#f8f8f8;"> <td>
<b>Side discussion:</b> Should "destination type" be mandatory or have a default?
<p/><p/>
We need to decide whether the application should be required to specify whether the destination is a queue or a topic.
<p/><p/> Currently EJB 3.2 is problematic: it defines an activation property <tt>destinationType</tt> but  does not define what should happen if it is not set: it's not described as mandatory but there's no default value. Are application server/resource adapters expected to define their own default, or are they expected to be able to work when this property is not set? If the latter, then what is the point of having this property?
<p/><p/>
For new-style MDBs we have four options:
<p/><p/>* Make it mandatory to set the destination type. This is the current proposal: the  <tt>@JMSListener</tt> attribute "<tt>type</tt>" is mandatory
<p/><p/>* Make it optional, but define a default value (queue or topic)
<p/><p/>* Make it optional, and leave it to application servers to choose a default (which will be non-portable)
<p/><p/>* Make it optional, and require application servers to be able to handle the case where it is omitted (which may be non-portable)
</td></tr></table>

===An alternative proposal (Option B)===

If we went for  a smaller number of annotations, each of which set multiple attributes, what might they look like? Here's a suggestion.

This proposals would replace the eight annotations of Option A with three new annotations:

* <tt>@JMSQueueListener</tt>

* <tt>@JMSDurableTopicListener</tt>

* <tt>@JMSNonDurableTopicListener</tt>

plus zero or more occurrences of the newly-proposed annotation:

* <tt>@JMSListenerProperty</tt>

This would remove the need for the user to specify destinationType or subscriptionDurability as attributes.

Here's an example of a MDB which listens for messages from a queue:
<p/>
 @JMSQueueListener(destinationLookup="java:global/java:global/Trades",
    connectionFactoryLookup="java:global/MyCF",
    messageSelector=""ticker='ORCL'",
    acknowledge=JMSQueueListener.Mode.DUPS_OK_ACKNOWLEDGE)
 @JMSListenerProperty(name="foo1", value="bar1")
 @JMSListenerProperty(name="foo2", value="bar2")
 public void giveMeAMessage(Message message) {
    ...
 }
<p/>
Here's an example of a MDB which listens for messages from a durable topic subscription:
<p/>
 @JMSDurableTopicListener(destinationLookup="java:global/java:global/Trades",
    connectionFactoryLookup="java:global/MyCF",
    clientId="myClientID1",
    subscriptionName="mySubName1",
    messageSelector=""ticker='ORCL'",
    acknowledge=JMSQueueListener.Mode.DUPS_OK_ACKNOWLEDGE)
 @JMSListenerProperty(name="foo1", value="bar1")
 @JMSListenerProperty(name="foo2", value="bar2")
 public void giveMeAMessage(Message message) {
    ...
 }
<p/>
Here's an example of a MDB which listens for messages from a non-durable topic subscription:
<p/>
 @JMSNonDurableTopicListener(destinationLookup="java:global/java:global/Trades",
    connectionFactoryLookup="java:global/MyCF",
    messageSelector=""ticker='ORCL'",
    acknowledge=JMSQueueListener.Mode.DUPS_OK_ACKNOWLEDGE)
 @JMSListenerProperty(name="foo1", value="bar1")
 @JMSListenerProperty(name="foo2", value="bar2")
 public void giveMeAMessage(Message message) {
    ...
 }
<p/>
Some of these attributes would need to be specified, others would be optional.

The main benefits of this option are:

* Fewer annotations for the user to remember, and more scope for the IDE to provide guidance. The user simply needs to choose between <tt>@JMSQueueListener</tt>, <tt>@JMSDurableTopicListener</tt> and <tt>@JMSNonDurableTopicListener</tt> and the IDE would then remind them which standard attributes that they may need to set.

* No need to explicitly specify destination type or subscription durability via verbose enumerated types

=== Option C===

A third option might be to combine option B with an additional, completely generic annotation.

In addition to <tt>@JMSQueueListener</tt>, <tt>@JMSDurableTopicListener</tt> and <tt>@JMSNonDurableTopicListener</tt>, there would also be an alternative, generic annotation  <tt>@JMSDestinationListener</tt>. This would strictly be unnecessary, but some users may prefer it:

 @JMSDestinationListener(destinationLookup="java:global/java:global/Trades",
    type=JMSListener.DestinationType.TOPIC,
    connectionFactoryLookup="java:global/MyCF",
    clientId="myClientID1",
    subscriptionDurability=JMSListener.SubscriptionDurability.DURABLE
    subscriptionName="mySubName1",
    messageSelector=""ticker='ORCL'",
    acknowledge=JMSQueueListener.Mode.DUPS_OK_ACKNOWLEDGE)
 @JMSListenerProperty(name="foo1", value="bar1")
 @JMSListenerProperty(name="foo2", value="bar2")
 public void giveMeAMessage(Message message) {
    ...
 }

This generic annotation would need  to allow the attributes <tt>destinationType</tt> and <tt>subscriptionDurability</tt> to be specified.

=== Option D===

A fourth option would be to offer just the generic annotation <tt>@JMSDestinationListener</tt>, plus <tt>@JMSListenerProperty</tt>
