Writer-Reader Layer
===================

The lower level Writer-Reader Layer of *eprosima Fast RTPS* provides a raw implementation of th RTPS protocol. It provides more control over the internals of the protocol than the Publisher-Subscriber layer. 
Advanced users can make use of this layer directly to gain more control over the functionality of the library.


Relation to the Publisher-Subscriber Layer
------------------------------------------

Elements of this layer map one-to-one with elements from the Publisher-Subscriber Layer, with a few additions. The following table shows the name correspondence between layers:

        +----------------------------+---------------------+
        | Publisher-Subscriber Layer | Writer-Reader Layer |
        +============================+=====================+
        |          Domain            |     RTPSDomain      |
        +----------------------------+---------------------+
        |        Participant         |   RTPSParticipant   |
        +----------------------------+---------------------+
        |         Publisher          |     RTPSWriter      |
        +----------------------------+---------------------+
        |         Subscriber         |     RTPSReader      |
        +----------------------------+---------------------+

How to use the Writer-Reader Layer 
----------------------------------

We will now go over the use of the Writer-Reader Layer like we did with the Publish-Subscriber one, explaining the new features it presents.

We recommend you to look at the two examples of how to use this layer the distribution comes with while reading this section. They are located in `examples/RTPSTest_as_socket` and in `examples/RTPSTest_registered`

Managing the Participant
^^^^^^^^^^^^^^^^^^^^^^^^

To create a RTPSParticipant, the process is very similar to the one shown in the Publisher-Subscriber layer. ::

        RTPSParticipantAttributes PParam;
        Pparam.setName("participant");
        RTPSParticipant* p = RTPSDomain::createRTPSParticipant(PParam);

The `RTPSParticipantAttributes` structure is equivalent to the `ParticipantAttributes.rtps` field in the Publisher-Subscriber Layer, so you can configure your `RTPSParticipant` the same way as before: ::

        RTPSParticipantAttributes Pparam;
        PParam.setName("my_participant");
        //etc.
	
Managing the Writers and Readers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As the RTPS standard specifies, Writers and Readers are always associated with a History element. In the Publisher-Subscriber Layer its creation and management is hidden, 
but in the Writer-Reader Layer you have full control over its creation and configuration.

Writers are configured with a `WriterAttributes` structure. They also need a `WriterHistory` which is configured with a `HistoryAttributes` structure. ::

        HistoryAttributes hatt;
        WriterHistory * history = new WriterHistory(hatt);
        WriterAttributes watt;
        RTPSWriter* writer = RTPSDomain::createRTPSWriter(rtpsParticipant,watt,hist);

The creation of a Reader is similar. Note that in this case you can provide a ReaderListener instance that implements your callbacks: ::

        class MyReaderListener:public ReaderListener;
        MyReaderListener listen;
        HistoryAttributes hatt;
        ReaderHistory * history = new ReaderHistory(hatt);
        ReaderAttributes ratt;
        RTPSReader* reader = RTPSDomain::createRTPSReader(rtpsParticipant,watt,hist,&listen);

Using the History to Send and Receive Data
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the RTPS Protocol, Readers and Writers save the data about a topic in their associated History. Each piece of data is represented by a Change, which *eprosima Fast RTPS* implements as `CacheChange_t`.
Changes are always managed by the History. As an user, the procedure for interacting with the History is always the same:

1. Request a `CacheChange_t` from the History
2. Use it
3. Release it
	
You can interact with the History of the Writer to send data: ::        

        CacheChange_t* ch = writer->newCacheChange(ALIVE);	//Request a change from the history
        ch->serializedPayload->length = sprintf(ch->serializedPayload->data,"My String %d",2);	//Write serialized data into the change
        history->add_change(ch);	//Insert change back into the history. The Writer takes care of the rest.

If your topic data type has several fields, you will have to provide functions to serialize and deserialize your data in and out of the `CacheChange_t`.
*FastRTPSGen* does this for you.
	
You can receive data from within a ReaderListener callback method as we did in the Publisher-Subscriber Layer: ::

        class MyReaderListener: public ReaderListener{
        	public:
        	MyReaderListener(){}
        	~MyReaderListener(){}
        	void onNewCacheChangeAdded(RTPSReader* reader,const CacheChange_t* const change)
        	{
        		printf("%s\n",change->serializedPayload.data);		// The incoming message is enclosed within the `change` in the function parameters
        		reader->getHistory()->remove_change((CacheChange_t*)change);	//Once done, remove the change
        	}
        }

Additionally you can read an incoming message directly by interacting with the History: ::

        reader->waitForUnreadMessage(); //Blocking method
        CacheChange_t* change;
        if(reader->nextUnreadCache(&change))	//Take the first unread change present in the History
        {
        	/* use data */
        }
        history->remove_change(change); //Once done, remove the change
	
Configuring Readers and Writers
-------------------------------
One of the benefits of using the Writer-Reader layer is that it provides new configuration possibilities while mainining the options from the Publisher-Subscriber layer.
For example, you can set a Writer or a Reader as a Reliable or Best-Effort endpoint as previously: ::

        Wattr.endpoint.reliabilityKind = BEST_EFFORT;

Setting the Input and Output Channels
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As in the Publisher-Subscriber Layer, you can specify the input and output channels you want your Writer/Reader to listen to or speak from in the form of Locators. 
This configuration overrides the one inherited from the `RTPSParticipant`. ::

        WriterAttributes Wattr;
        Locator_t my_locator;
        //Set up your Locator
        Wattr.endpoint.OutLocatorList.push_back(my_locator);
	
Setting the data durability kind
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Durability parameter defines the behaviour of the Writer regarding samples already sent when a new Reader matches. *eProsima Fast RTPS* offers two Durability options:

* VOLATILE (default): Messages are discarded as they are sent. If a new Reader matches after message *n*, it will start received from message *n+1*.
* TRANSIENT_LOCAL: The Writer saves a record of the lask *k* messages it has sent. If a new reader matches after message *n*, it will start receiving from message *n-k*

To choose you preferred option: ::

        WriterAttributes Wparams;
        Wparams.endpoint.durabilityKind = TRANSIENT_LOCAL;

Because in the Writer-Reader layer you have control over the History,in TRANSIENT_LOCAL mode the Writer send all changes you have not explicitly released from the History.

Configuring the History
-----------------------

The History has its own configuration structure, the `HistoryAttributes`.
	
Changing the maximum size of the payload
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can choose the maximum size of they Payload that can go into a CacheChange_t. Be sure to choose a size that allows it to hold the biggest extected piece of data: ::
	
        HistoryAttributes.payloadMaxSize  =	 250;	//Defaults to 500 bytes

Changing the size of the History
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can specify a maximum amount of changes for the History to hold and and initial amount of allocated changes: ::

        HistoryAttributes.initialReservedCaches = 250; //Defaults to 500
        HistoryAttributes.maximumReservedCaches = 500; //Dedaults to 0 = Unlimited Changes

When the initial amount of reserved changes is lower than the maximum, the History will allocate more changes as they are needed until it reaches the maximum size.
