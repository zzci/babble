.. _api:

API
===

Babble communicates with the App through an `AppProxy` interface, which has two
implementations:

- ``InmemProxy``: An InmemProxy uses native callback handlers to integrate
  Babble as a regular Go dependency.

- ``SocketProxy``: A SocketProxy connects to an App via TCP sockets. It enables
  the application to run in a separate process or machine, and to
  be written in any programming language.

The ``AppProxy`` interface exposes four methods for Babble to call the App:

- ``CommitBlock(block hashgraph.Block) (CommitResponse, error)``: Commits a 
  block to the application and returns the resulting state hash.

- ``OnStateChanged(state.State) error``: Notifies the app when Babble enters a 
  new internal state.
  
- ``GetSnapshot(int) ([]byte, error)``: Gets the application snapshot
  corresponding to a particular block index.

- ``Restore([]byte) error``: Restores the App state from a snapshot.

**Note**: The Snapshot and Restore methods of the API are still work in
progress. They are necessary for the :ref:`fastsync` protocol which is not
completely ready yet. It is safe to just implement stubs for these methods.

Reciprocally, ``AppProxy`` relays transactions from the App to Babble via a
native Go channel - ``SubmitCh`` - which ties into the application differently
depending on the type of proxy (Socket or Inmem).

Babble asynchronously processes transactions and eventually feeds them back to
the App, in consensus order and bundled into blocks, with a **CommitBlock**
call. Transactions are just raw bytes and Babble does not need to know what
they represent. Therefore, encoding and decoding transactions is done by the
App.

Inmem
-----

The ``InmemProxy`` uses native callback handlers to enable Babble to call
methods on the App directly. Applications need only implement the
``ProxyHandler`` interface and pass that to an ``InmemProxy``.

Here is a quick example of how to use Babble as an in-memory engine (in the
same process as your handler):

.. code:: go

    package main

    import (
    	"os"
    
    	"github.com/mosaicnetworks/babble/src/config"
    	"github.com/mosaicnetworks/babble/src/hashgraph"
    	"github.com/mosaicnetworks/babble/src/node/state"
    	"github.com/mosaicnetworks/babble/src/proxy"
    	"github.com/mosaicnetworks/babble/src/proxy/inmem"
    )
    
    // ExampleHandler implements the ProxyHandler interface. This is where an
    // application would normally register callbacks that Babble will call through
    // the InmemProxy. ExampleHandler simply maintains a list of all the committed
    // transactions in the order they were received from Babble, and keeps track of
    // Babble's state. Refer to the dummy package for a more meaningful example.
    type ExampleHandler struct {
    	transactions [][]byte
    	state        state.State
    }
    
    // CommitHandler is called by Babble to commit a block to the application.
    // Blocks contain transactions that represent commands for the application, and
    // internal transactions that are used internally by Babble to update the
    // peer-set. The application can accept or refuse internal transactions based on
    // custom rules. Here we accept all internal transactions. The commit response
    // contains a state hash that should represent the state of the application
    // after applying all the transactions sequentially.
    func (p *ExampleHandler) CommitHandler(block hashgraph.Block) (proxy.CommitResponse, error) {
    	// block transactions are ordered. Every Babble node will receive the same
    	// transactions in the same order.
    	p.transactions = append(p.transactions, block.Transactions()...)
    
    	// internal transactions represent requests to add or remove participants
    	// from the Babble peer-set. This decision can be based on the application
    	// state. For example the application could maintain a whitelist such that
    	// only people whose public key belongs to the whitelist will be accepted to
    	// join the peer-set. The decision must be deterministic, this is not a vote
    	// where every one gives their opinion. All peers must return the same
    	// answer, or risk creating a fork.
    	receipts := []hashgraph.InternalTransactionReceipt{}
    	for _, it := range block.InternalTransactions() {
    		receipts = append(receipts, it.AsAccepted())
    	}
    
    	// The commit response contains the state-hash resulting from applying all
    	// the transactions, and all the transaction receipts. Here we always
    	// return the same hard-coded state-hash.
    	response := proxy.CommitResponse{
    		StateHash:                   []byte("statehash"),
    		InternalTransactionReceipts: receipts,
    	}
    
    	return response, nil
    }
    
    // StateChangedHandler is called by Babble to notify the application that the
    // node has entered a new state (ex Babbling, Joining, Suspended, etc.).
    func (p *ExampleHandler) StateChangeHandler(state state.State) error {
    	p.state = state
    	return nil
    }
    
    // SnapshotHandler is used by Babble to retrieve a snapshot of the application
    // corresponding to a specific block index. It is left to the application to
    // keep track of snapshots and to encode/decode state snapshots to and from raw
    // bytes. This handler is only used when fast-sync is activated.
    func (p *ExampleHandler) SnapshotHandler(blockIndex int) ([]byte, error) {
    	return []byte("snapshot"), nil
    }
    
    // RestoreHandler is called by Babble to instruct the application to restore its
    // state back to a given snapshot. This is only used when fast-sync is
    // activated.
    func (p *ExampleHandler) RestoreHandler(snapshot []byte) ([]byte, error) {
    	return []byte("statehash"), nil
    }
    
    func NewExampleHandler() *ExampleHandler {
    	return &ExampleHandler{
    		transactions: [][]byte{},
    	}
    }
    
    func main() {
    	// An application needs to implement the ProxyHandler interface and define
    	// the callbacks that will be automatically called by the proxy when Babble
    	// has things to communicate to the application.
    	handler := NewExampleHandler()
    
    	// We create an InmemProxy based on the handler.
    	proxy := inmem.NewInmemProxy(handler, nil)
    
    	// Start from default configuration.
    	babbleConfig := config.NewDefaultConfig()
    
    	// Set the AppProxy in the Babble configuration.
    	babbleConfig.Proxy = proxy
    
    	// Instantiate Babble.
    	babble := NewBabble(babbleConfig)
    
    	// Read in the confiuration and initialise the node accordingly.
    	if err := babble.Init(); err != nil {
    		babbleConfig.Logger().Error("Cannot initialize babble:", err)
    		os.Exit(1)
    	}
    
    	// The application can submit transactions to Babble using the proxy's
    	// SubmitTx. Babble will broadcast the transactions to other nodes, run
    	// them through the consensus algorithm, and eventually call the callback
    	// methods implemented in the handler.
    	go func() {
    		proxy.SubmitTx([]byte("the test transaction"))
    	}()
    
    	// Run the node aynchronously.
    	babble.Run()
    
    	// Babble reacts to SIGINT (Ctrl + c) and SIGTERM by calling the leave
    	// method to politely leave a Babble network, but it can also be called
    	// manually.
    	defer babble.Node.Leave()
    }

Socket
------

The ``SocketProxy`` is simply a TCP server that accepts `SubmitTx` requests,
and calls remote methods on the App through a JSON-RPC interface. The App is
therefore expected to implement its own component to send out SubmitTx requests
through TCP, and receive JSON-RPC messages from the remote Babble node.

The advantage of using a TCP interface is that it provides the freedom to
implement the application in any programming language. The specification of the
JSON-RPC interface is provided below, but here is an example of how to use our
Go implementation, ``SocketBabbleProxy``, to connect to a remote Babble node.

Please refer to the dummy package for an example implementing the socket 
interface.
