godbus-connection-monitor is handy library that can be used by a GO application to continuously monitor the status of a dbus connection and be notified when that connection is lost.
To use this library, import the dbus_monitor package into your application, set up the dbus connection to either the system or session bus as you normally would and then pass the
dbus connection object to the library alongwith a callback function. When the connection to the bus is lost for any reason, the library will invoke the callback, at which stage,
the application can either reinitiate the connection to the bus or execute other appropriate error handling. Here is an example

// Define a callback function to be invoked when the connection to dbus is lost
func onDbusConnectionLost(conn *dbus.Conn, err error) {
  fmt.Println("Connection to dbus lost due to error: ", err.Error())
}
..... elsewhere in your application

// Connect to the system bus
conn, err := dbus.SystemBus()

// Add Match signals, export objects, request names etc
reply, err := conn.RequestName("myservice", dbus.NameFlagDoNotQueue)

// Register this dbus connection to be monitored by the library
dbus_monitor.MonitorConnection(conn, onDbusConnectionLost, dbusMonitor.DefaultOptions())

The dbus_monitor internally periodically checks the health of the connection to the bus by invoking the Ping() method on the org.freedesktop.DBus.Peer interface.
The checking behavior is controlled by a ping interval and a ping timeout. Default value for the ping interval is 5 seconds and that for the ping timeout is 3 seconds
These can be changed by the application and then passed to the dbus_monitor with the following code

opts := dbus_monitor.MonitorOptions{
  PingIntervalS: 10 * time.Second,
  PingTimeoutS: 6 * time.Second,
}

dbus_monitor.MonitorConnection(conn, onDbusConnectionLost, &opts)

Naturally, the PingIntervalS > PingTimeoutS, otherwise the MonitorConnection method will return an error.

An application can register multiple dbus connection objects to be monitored by making use of the MultiMonitor object. This can be used
as follows

multiMonitor := dbus_monitor.NewMultiMonitor()

// Add system bus with aggressive monitoring
systemConn, err := dbus.ConnectSystemBus()
defer systemConn.Close()

systemOpts := dbusmonitor.MonitorOptions{
  PingInterval: 2 * time.Second,
	PingTimeout:  1 * time.Second,
}
	
systemCallback := func(conn *dbus.Conn, err error) {
	log.Printf("System bus connection lost: %v", err)
	// Handle system bus reconnection
}

if err := multiMonitor.AddConnection(systemConn, systemCallback, &systemOpts); err != nil {
	log.Fatalf("Failed to add system connection: %v", err)
}

// Add session bus with relaxed monitoring
sessionConn, err := dbus.ConnectSessionBus()
if err != nil {
	log.Fatalf("Failed to connect to session bus: %v", err)
}
defer sessionConn.Close()

sessionOpts := dbusmonitor.MonitorOptions{
		PingInterval: 10 * time.Second,
		PingTimeout:  5 * time.Second,
	}
	
sessionCallback := func(conn *dbus.Conn, err error) {
	log.Printf("Session bus connection lost: %v", err)
	// Handle session bus reconnection
}

if err := multiMonitor.AddConnection(sessionConn, sessionCallback, &sessionOpts); err != nil {
	log.Fatalf("Failed to add session connection: %v", err)
}
