# vertica_rust_connection
Documents basic configuration needed to connect to a Vertica DB using Rust

## Reference

[Connecting Vertica to ODBC](https://docs.vertica.com/23.4.x/en/connecting-to/client-libraries/client-drivers/install-config/odbc/)

## Tested on

This was tested in the Rust dev container of Visual Studio Code

```shell
$ cat /etc/os-release 
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

## Getting the driver

```bash
sudo wget https://www.vertica.com/client_drivers/12.0.x/12.0.4-0/vertica-client-12.0.4-0.x86_64.tar.gz
```

## Installing the driver

The final location of the driver is `/opt/vertica`. The version you get depends on your Vertica release.

```bash
sudo tar zxvf /path/to/vertica-client-12.0.4-0.x86_64.tar.gz -C /
```

After the extraction:

```txt
├── opt
│   └── vertica
```

## Required packages

```bash
sudo apt-get update
sudo apt-get install unixodbc unixodbc-dev
```

## Configuration

You need to configure 

**/etc/vertica.ini**

```ini
[Driver]
DriverManagerEncoding=UTF-16
ODBCInstLib=/usr/lib/x86_64-linux-gnu/libodbcinst.so.2
ErrorMessagesPath=/opt/vertica/lib64
LogLevel=4
LogPath=/tmp
```

**/etc/odbc.ini**

```ini
[ODBC Data Sources]
Dev1Vertica="nBA database on Vertica Dev1"

[Dev1Vertica]
Description=Vertica Database
Driver=/opt/vertica/lib64/libverticaodbc.so
Database=defaultdb
Server=1.1.1.1
UID=ro_user
PWD=password
Port=5433
```

**/etc/odbcinst.ini**

```ini
[Vertica]
Description=Vertica ODBC Driver
Driver=/opt/vertica/lib64/libverticaodbc.so
UsageCount=1
[ODBC]
Threading=1 
Trace=Yes
TraceFile=/opt/vertica/odbc_trace.log
```
**Environment Variables**

Load them in your shell rc file:

```text
export LD_LIBRARY_PATH=/opt/vertica/lib64:$LD_LIBRARY_PATH
export PATH=/opt/vertica/bin:$PATH
export VERTICAINI=/etc/vertica.ini
export ODBCSYSINI=/etc
export ODBCINI=/etc/odbc.ini
export ODBCINSTINI=/etc/odbcinst.ini
```

For bashrc:

```shell
vi ~/.bashrc
source ~/.bashrc
```

## Check unixODBC Configuration:

Use the odbcinst command-line tool to check the configuration:
```bash
odbcinst -j
```

```shell
$ odbcinst -j
unixODBC 2.3.6
DRIVERS............: /etc/
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /home/vscode/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

This command will print the locations of the odbc.ini and odbcinst.ini files as recognized by unixODBC.

## Check DSN Listing

```bash
odbcinst -q -s
```

Output:

```shell
$ odbcinst -q -s
[Dev1Vertica]
```

## Check Driver Listing

```bash
odbcinst -q -d
```

Never got this one to work kept getting:

```shell
$ odbcinst -q -d
odbcinst: SQLGetPrivateProfileString failed with Unable to find component name.
```

## The Crate and Code

The 


[odbc_api](https://crates.io/crates/odbc-api)

Example taken directly from the crate documentation:

```Rust
//! A program executing a query and printing the result as csv to standard out. Requires
//! `anyhow` and `csv` crate.

use anyhow::Error;
use odbc_api::{buffers::TextRowSet, Cursor, Environment, ConnectionOptions, ResultSetMetadata};
use std::io::stdout;

/// Maximum number of rows fetched with one row set. Fetching batches of rows is usually much
/// faster than fetching individual rows.
const BATCH_SIZE: usize = 5000;

fn main() -> Result<(), Error> {

    // Write csv to standard out
    let out = stdout();
    let mut writer = csv::Writer::from_writer(out);
    
    // If you do not do anything fancy it is recommended to have only one Environment in the
    // entire process.
    let environment = Environment::new()?;
    for driver_info in environment.drivers()? {
        print!("Driver: ");
        println!("{:#?}", driver_info); 
    }
    for data_source in environment.data_sources()? {
        print!("Data Source: ");
        println!("{:#?}", data_source);
    }

    // Connect using a DSN. Alternatively we could have used a connection string
    let connection = environment.connect(
        "Dev1Vertica",
        "ro_user",
        "password",
        ConnectionOptions::default(),
    )?;

    // Execute a one of query without any parameters.
    match connection.execute("SELECT * FROM schema.table", ())? {
        Some(mut cursor) => {
            // Write the column names to stdout
            let headline : Vec<String> = cursor.column_names()?.collect::<Result<_,_>>()?;
            writer.write_record(headline)?;

            // Use schema in cursor to initialize a text buffer large enough to hold the largest
            // possible strings for each column up to an upper limit of 4KiB.
            let mut buffers = TextRowSet::for_cursor(BATCH_SIZE, &mut cursor, Some(4096))?;
            // Bind the buffer to the cursor. It is now being filled with every call to fetch.
            let mut row_set_cursor = cursor.bind_buffer(&mut buffers)?;

            // Iterate over batches
            while let Some(batch) = row_set_cursor.fetch()? {
                // Within a batch, iterate over every row
                for row_index in 0..batch.num_rows() {
                    // Within a row iterate over every column
                    let record = (0..batch.num_cols()).map(|col_index| {
                        batch
                            .at(col_index, row_index)
                            .unwrap_or(&[])
                    });
                    // Writes row as csv
                    writer.write_record(record)?;
                }
            }
        }
        None => {
            eprintln!(
                "Query came back empty. No output has been created."
            );
        }
    }

    Ok(())
}
```


