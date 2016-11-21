# clustersql froked and allow multi clustered datasource
[![GoDoc](https://godoc.org/github.com/EnumApps/clustersql?status.svg)](http://godoc.org/github.com/EnumApps/clustersql)


Go Clustering SQL Driver - A clustering, implementation-agnostic "meta"-driver for any backend implementing "database/sql/driver".

It does (latency-based) load-balancing and error-recovery over all registered nodes.

**It is assumed that database-state is transparently replicated over all nodes by some database-side clustering solution. This driver ONLY handles the client side of such a cluster.**

This package simply multiplexes the driver.Open() function of sql/driver to every attached node. The function is called on each node, returning the first successfully opened connection. (Any connections opening subsequently will be closed.) If opening does not succeed for any node, the latest error gets returned. Any other errors will be masked by default. However, any given latest error for any attached node will remain exposed through expvar, as well as some basic counters and timestamps.
    
To make use of this kind of clustering, use this package with any backend driver implementing "database/sql/driver" like so:

	import "database/sql"
	import "github.com/go-sql-driver/mysql"
	import "github.com/EnumApps/clustersql"

const (
	WriteDriver = "write_conn"
	ReadDriver = "read_conn"
	SessDriver = "sess_conn"
)
There is currently no way around instanciating the backend driver explicitly

	mysqlDriver := mysql.MySQLDriver{}

You can perform backend-driver specific settings such as

	err := mysql.SetLogger(mylogger)

Create a new clustering driver with the backend driver

	readerDriver := clustersql.NewDriver(mysqlDriver, ReadDriver)

Add nodes, including driver-specific name format, in this case Go-MySQL DSN. Here, we add three nodes belonging to a [galera](https://mariadb.com/kb/en/mariadb/documentation/replication-cluster-multi-master/galera/) cluster

	readerDriver.AddNode("galera1", "reader:password@tcp(dbhost1:3306)/db")
	readerDriver.AddNode("galera2", "reader:password@tcp(dbhost2:3306)/db")
	readerDriver.AddNode("galera3", "reader:password@tcp(dbhost3:3306)/db")

Make the clusterDriver available to the go sql interface under an arbitrary name

	sql.Register(ReadDriver, readerDriver)

Create a new clustering driver with the backend driver

	sessionDriver := clustersql.NewDriver(mysqlDriver, SessDriver)

Add nodes, including driver-specific name format, in this case Go-MySQL DSN. Here, we add three nodes belonging to a [galera](https://mariadb.com/kb/en/mariadb/documentation/replication-cluster-multi-master/galera/) cluster

	sessionDriver.AddNode("galera1", "sess_user:password@tcp(dbhost1:3306)/sessdb")
	sessionDriver.AddNode("galera2", "sess_user:password@tcp(dbhost2:3306)/sessdb")
	sessionDriver.AddNode("galera3", "sess_user:password@tcp(dbhost3:3306)/sessdb")

Make the clusterDriver available to the go sql interface under an arbitrary name

	sql.Register(SessDriver, sessionDriver)



Open the registered clusterDriver with an arbitrary DSN string (not used)

	db, err := sql.Open(WriteDriver, "")

	readonly_db, err := sql.Open(ReadDriver, "")

	session_db, err := sql.Open(SessDriver, "")

Continue to use the sql interface as documented at http://golang.org/pkg/database/sql/


Before using this in production, you should configure your cluster details in config.toml and run

    go test -v .

Note however, that non-failure of the above is no guarantee for a correctly set-up cluster.
