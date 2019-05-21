# master-corda-example
Corda example for Master's work. Based [on Corda tutorial](https://github.com/corda/cordapp-template-java)
Development made on MacOS v10.14 with IntelliJ IDEA IDE (no need to download for testing purposes - Terminal would be enough).

# Description
Each IOU – short for “I O(we) (yo)U” – will record the fact that one node owes another node a certain amount. 
The example CorDapp allows nodes to agree IOUs with each other, as long as they obey the following contract rules:
* The IOU’s value is strictly positive
* the A node is not trying to issue an IOU to itself
Because data is only propagated on a need-to-know basis, any IOUs agreed between PartyA and PartyB become “shared facts” between PartyA and PartyB only. PartyC won’t be aware of these IOUs.

# Installation Procedure
To implement an Corda smart contract for a local blockchain nodes, you need the following tools:
[Java 8 JVM](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) - required at least version 8u171 (currently no support for Java 9 or higher).
Gradle 4.10 - the **gradlew** script in the project directories will download it for you during deployment phase

## Compile and Deployment
First of all need to clone Corda example to local directory (either through Git or download .zip file from GitHub)
```
git clone https://github.com/wonderbeak/master-corda-example.git
cd master-corda-example
```

To compile the project just run following task with Gradle script
```
./gradlew compileKotlin
```
This command will deploy the CorDapp on 3 test nodes that may be found under "build/nodes/:
* Notary, which runs a notary service (name = "O=Notary,L=London,C=GB")
* PartyA (name = "O=PartyA,L=London,C=GB")
* PartyB (name = "O=PartyB,L=New York,C=US")
```
./gradlew deployNodes
```

![deployNodes](https://imgur.com/download/nneCX3R)

To run this nodes there is need to execute **runnodes** command with deployed nodes from previous command:
```
cd build/nodes/
./runnodes
```
![runnodes](https://imgur.com/download/wzegydx)

After successful running 3 new Terminals will be opened - for Notary, PartyA and PartyB. Notary node:
![Notary node](https://imgur.com/download/4qqZMkO)

**To close the nodes - type "bye" or "exit" in every Terminal**

# Tests

## Automatic
To run automatic tests just type this command in project directory:
```
./gradlew test
```
This will test deploying nodes and dummy flows.

## Manual
To perform some transactions just open in Terminal any node PartyA or PartyB and input appropriate details in this query (just change the value and the party name found above - DO NOT try with Notary node)
```
start IOUFlow iouValue: 13, otherParty: "O=PartyB,L=New York,C=US"
```

![IOUFlow](https://imgur.com/download/jnBhz8I)

To check this query result just input on the same Terminal where previous command was executed
```
run vaultQuery contractStateType: com.template.IOUState
```

Following output will be shown: 
```
{
  "states" : [ {
    "state" : {
      "data" : {
        "value" : 13,
        "lender" : "O=PartyA, L=London, C=GB",
        "borrower" : "O=PartyB, L=New York, C=US",
        "participants" : [ "O=PartyA, L=London, C=GB", "O=PartyB, L=New York, C=US" ]
      },
      "contract" : "com.template.IOUContract",
      "notary" : "O=Notary, L=London, C=GB",
      "encumbrance" : null,
      "constraint" : { }
    },
    "ref" : {
      "txhash" : "FF62D94B9631279A80D35E139029B1901FB993011984D14C615515FB66C88C24",
      "index" : 0
    }
  } ],
  "statesMetadata" : [ {
    "ref" : {
      "txhash" : "FF62D94B9631279A80D35E139029B1901FB993011984D14C615515FB66C88C24",
      "index" : 0
    },
    "contractStateClassName" : "com.template.IOUState",
    "recordedTime" : "2019-05-21T15:35:28.940Z",
    "consumedTime" : null,
    "status" : "UNCONSUMED",
    "notary" : "O=Notary, L=London, C=GB",
    "lockId" : null,
    "lockUpdateTime" : null
  } ],
  "totalStatesAvailable" : -1,
  "stateTypes" : "UNCONSUMED",
  "otherResults" : [ ]
}
```

# Contract, State and Flows
```
// Contract
class IOUContract : Contract {
    companion object {
        const val ID = "com.template.IOUContract"
    }

    // Create command.
    class Create : CommandData

    override fun verify(tx: LedgerTransaction) {
        val command = tx.commands.requireSingleCommand<Create>()

        requireThat {
            // Constraints on the shape of the transaction.
            "No inputs should be consumed when issuing an IOU." using (tx.inputs.isEmpty())
            "There should be one output state of type IOUState." using (tx.outputs.size == 1)

            // IOU-specific constraints.
            val output = tx.outputsOfType<IOUState>().single()
            "The IOU's value must be non-negative." using (output.value > 0)
            "The lender and the borrower cannot be the same entity." using (output.lender != output.borrower)

            // Constraints on the signers.
            val expectedSigners = listOf(output.borrower.owningKey, output.lender.owningKey)
            "There must be two signers." using (command.signers.toSet().size == 2)
            "The borrower and lender must be signers." using (command.signers.containsAll(expectedSigners))
        }
    }
}

// State
class IOUState(val value: Int,
               val lender: Party,
               val borrower: Party) : ContractState {
    override val participants get() = listOf(lender, borrower)
}
```

Flows
```
@InitiatingFlow
@StartableByRPC
class IOUFlow(val iouValue: Int,
              val otherParty: Party) : FlowLogic<Unit>() {

    //The progress tracker provides checkpoints indicating the progress of the flow to observers.
    override val progressTracker = ProgressTracker()

    //The flow logic is encapsulated within the call() method.
    @Suspendable
    override fun call() {
        // We retrieve the notary identity from the network map.
        val notary = serviceHub.networkMapCache.notaryIdentities[0]

        // We create the transaction components.
        val outputState = IOUState(iouValue, ourIdentity, otherParty)
        val command = Command(IOUContract.Create(), listOf(ourIdentity.owningKey, otherParty.owningKey))

        // We create a transaction builder and add the components.
        val txBuilder = TransactionBuilder(notary = notary)
                .addOutputState(outputState, IOUContract.ID)
                .addCommand(command)

        // Verifying the transaction.
        txBuilder.verify(serviceHub)

        // Signing the transaction.
        val signedTx = serviceHub.signInitialTransaction(txBuilder)

        // Creating a session with the other party.
        val otherPartySession = initiateFlow(otherParty)

        // Obtaining the counterparty's signature.
        val fullySignedTx = subFlow(CollectSignaturesFlow(signedTx, listOf(otherPartySession), CollectSignaturesFlow.tracker()))

        // Finalising the transaction.
        subFlow(FinalityFlow(fullySignedTx))
    }
}

@InitiatedBy(IOUFlow::class)
class IOUFlowResponder(val otherPartySession: FlowSession) : FlowLogic<Unit>() {
    @Suspendable
    override fun call() {
        val signTransactionFlow = object : SignTransactionFlow(otherPartySession, SignTransactionFlow.tracker()) {
            override fun checkTransaction(stx: SignedTransaction) = requireThat {
                val output = stx.tx.outputs.single().data
                "This must be an IOU transaction." using (output is IOUState)
                val iou = output as IOUState
                "The IOU's value can't be too high." using (iou.value < 100)
            }
        }

        subFlow(signTransactionFlow)
    }
}
```
