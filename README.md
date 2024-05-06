# go-nfdump

[![Go Reference](https://pkg.go.dev/badge/github.com/phaag/go-nfdump.svg)](https://pkg.go.dev/github.com/phaag/go-nfdump)
[![buildtest](https://github.com/phaag/go-nfdump/actions/workflows/go.yml/badge.svg)](https://github.com/phaag/go-nfdump/actions/workflows/go.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/phaag/go-nfdump)](https://goreportcard.com/report/github.com/phaag/go-nfdump)

This Go module allows to read and process files created by [nfdump](https://github.com/phaag/nfdump), the netflow/ipfix/sflow collector and processing tools.

This module is experimental and does not yet decode all available nfdump record extensions. It reads and processes only nfdump v2 files, which are created by nfdump-1.7.x. Files created with nfdump-1.6.x are recogized but skipped for decoding.

Expample to read and process a flow file:



```go

package main

import (
	"flag"
	"fmt"
	"os"

	nfdump "github.com/phaag/go-nfdump"
)

var (
	fileName = flag.String("r", "", "nfdump file to read")
)

func main() {

	flag.CommandLine.Usage = func() {
		fmt.Fprintf(os.Stderr, "Usage of %s [flags]\n", os.Args[0])
		flag.PrintDefaults()
	}

	flag.Parse()

	if len(*fileName) == 0 {
		fmt.Printf("Filename required\n")
		flag.PrintDefaults()
		os.Exit(255)
	}

	nffile := nfdump.New()

	if err := nffile.Open(*fileName); err != nil {
		fmt.Printf("Failed to open nf file: %v\n", err)
		os.Exit(255)
	}

	// print nffile stats
	fmt.Printf("nffile:\n%v", nffile)

	// Dump flow records
	recordChannel, _ := nffile.AllRecords()
	cnt := 0
	for record := range recordChannel {
		cnt++

		// check IP addresses in record for IPv4, or IPv6
		if record.IsIPv4() {
			fmt.Printf("Record %d is IPv4\n", cnt)
		} else if record.IsIPv6() {
			fmt.Printf("Record %d is IPv6\n", cnt)
		} else {
			fmt.Printf("Record %d has no IPs\n", cnt)
		}

		// sampling
 		packetInterval, spaceInterval := record.SamplerInfo(nffile)
		fmt.Printf("Record sampler info: packet interval: %d, space interval: %d\n",
               packetInterval, spaceInterval)
    
		// print the entire record using %v
		fmt.Printf("%v\n", record)

		// get generic extension and print ports
		// see nfxV3.go for all fields in genericFlow
		if genericFlow := record.GenericFlow(); genericFlow != nil {
			fmt.Printf("SrcPort: %d\n", genericFlow.SrcPort)
			fmt.Printf("DstPort: %d\n", genericFlow.DstPort)
		}

		// get src, dst ip address extension of record
		// can contain IPv4 or IPv6
		ipAddr := record.IP()
		if ipAddr != nil {
			// when printing as %v, Golang takes care about proper formating
			// as IPv4 or IPv6
			// see Golang standard library net.IP for more details to process IPs
			fmt.Printf("SrcIP: %v\n", ipAddr.SrcIP)
			fmt.Printf("DstIP: %v\n", ipAddr.DstIP)
		}
    
    // get NAT xlate IP adresses
    if natXlateIP = flowRecord.NatXlateIP(); natXlateIP != nil {
      fmt.Sprintf("  SrcXlateIP  : %v\n", natXlateIP.SrcXIP)
      fmt.Sprintf("  DstXlateIP  : %v\n", natXlateIP.DstXIP)
    }
    
    // get NAT xlate ports
    if natXlatePort := flowRecord.NatXlatePort(); natXlatePort == nil {
      fmt.Printf("  Src X-Port  : %d\n", natXlatePort.XlateSrcPort)
      fmt.Printf("  Dst X-Port  : %d\n", natXlatePort.XlateDstPort)
    }
  
    // get nat port block and print
    if natPortBlock := flowRecord.NatPortBlock(); natPortBlock == nil {
      fmt.Printf("  NAT pstart  : %d\n", natPortBlock.BlockStart)
			fmt.Printf("  NAT pend    : %d\n", natPortBlock.BlockEnd)
			fmt.Printf("  NAT pstep   : %d\n", natPortBlock.BlockStep)
			fmt.Printf("  NAT psize   : %d\n", natPortBlock.BlockSize)
		}
   
		/*
			// other extension
			// see nfxV3.go for all fields in the respectiv records
			// always check for nil return value as not every extension
			// is available
			flowMisc := record.FlowMisc()
			cntFlow := record.CntFlow()
			vLan := record.VLan()
			asRouting := record.AsRouting()
			bgpNextHop := record.BgpNextHop()
			ipNextHop := record.IpNextHop()
			
			// please note, sampling contains only references to exporter list
			// use record.SamplerInfo(nffile) to retrieve true sampling values
			sampling := record.Sampling()
		*/
	}
  
	// retrieve exporter list *after* all records are processed
	exporterList := nffile.GetExporterList()
	fmt.Printf("Exporter list:\n")
	for id, exporter := range exporterList {
		if exporter.IP != nil && id == int(exporter.SysId) { // valid exporter
			fmt.Printf("  SysID: %d, ID: %d, IP: %v, version: %d", 
                 exporter.SysId, exporter.Id, exporter.IP, exporter.Version)
			fmt.Printf(" Sequence failures: %d, packets: %d, flows: %d\n",
                 exporter.SequenceFailures, exporter.Packets, exporter.Flows)
		}
	}
}
```

The `defs.go` file includes nfdump's `nfxV3.h` header file to convert individual record extensions into appropriate Golang records. So far the generic, misc, flowCount, vlan and asRouting extensions as well as IPv4/IPv6 addresses are available through the interface. See the nfxV3.go file for its definitions.

If you modify the `defs.go` file, generate `nfxV3.go` use the go command

`go generate ./...`

All available extensions are visible in `nfxV3.go`. 

Please note, that the interface may be subject to change, as this module is work in progress.

More element data blocks will follow, including the famous nfdump filter engine.
Please submit your pull requests and/or bug reports via [GitHub](https://github.com/phaag/go-nfdump/issues).
