package main

import (
	"bufio"
	"flag"
	"fmt"
	"os"
	"regexp"
	"sort"
	"strconv"
)

// This simple analyzer supports common combined-format-like lines and extracts
// status code and request path. It summarizes top endpoints and status counts.

var lineRegex = regexp.MustCompile(`"(?P<method>[A-Z]+) (?P<path>[^ ]+) HTTP/[^"]+" (?P<status>\d{3}) (?P<size>\d+)`)

func main() {
	path := flag.String("log", "", "path to log file (required)")
	top := flag.Int("top", 10, "show top N endpoints")
	flag.Parse()
	if *path == "" {
		fmt.Println("Usage: -log=access.log")
		os.Exit(1)
	}

	f, err := os.Open(*path)
	if err != nil {
		fmt.Println("open error:", err)
		return
	}
	defer f.Close()

	statusCounts := make(map[string]int)
	endpointCounts := make(map[string]int)
	scanner := bufio.NewScanner(f)
	var total int
	for scanner.Scan() {
		line := scanner.Text()
		total++
		m := lineRegex.FindStringSubmatch(line)
		if m == nil {
			continue
		}
		path := m[2]
		status := m[3]
		statusCounts[status]++
		endpointCounts[path]++
	}
	if err := scanner.Err(); err != nil {
		fmt.Println("scan error:", err)
	}
	fmt.Println("Log Analyzer Summary")
	fmt.Println("--------------------")
	fmt.Printf("Total lines scanned: %d\n", total)
	fmt.Println("\nStatus codes:")
	for s, c := range statusCounts {
		fmt.Printf(" %s: %d\n", s, c)
	}

	// top endpoints
	type kv struct {
		K string
		V int
	}
	var arr []kv
	for k, v := range endpointCounts {
		arr = append(arr, kv{k, v})
	}
	sort.Slice(arr, func(i, j int) bool { return arr[i].V > arr[j].V })
	fmt.Printf("\nTop %d endpoints:\n", *top)
	for i := 0; i < len(arr) && i < *top; i++ {
		fmt.Printf(" %d) %s — %d\n", i+1, arr[i].K, arr[i].V)
	}

	// quick error rate
	var errCount int
	for code, cnt := range statusCounts {
		c, _ := strconv.Atoi(code)
		if c >= 400 {
			errCount += cnt
		}
	}
	fmt.Printf("\nErrors (4xx/5xx): %d (%.2f%%)\n", errCount, float64(errCount)/float64(total)*100)
}
