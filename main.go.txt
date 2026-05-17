package main

import (
	"crypto/tls"
	"flag"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"regexp"
	"strings"
	"sync"
	"time"
)

// =====================
// ANSI Colors
// =====================
const (
	colorReset  = "\033[0m"
	colorRed    = "\033[31m"
	colorGreen  = "\033[32m"
	colorYellow = "\033[33m"
	colorCyan   = "\033[36m"
	colorBold   = "\033[1m"
)

// =====================
// Social Media Patterns
// =====================
var socialPatterns = map[string]*regexp.Regexp{
	"LinkedIn":  regexp.MustCompile(`https?://(?:www\.)?linkedin\.com/in/([a-zA-Z0-9\-_%]+)`),
	"Twitter/X": regexp.MustCompile(`https?://(?:www\.)?(?:twitter\.com|x\.com)/([a-zA-Z0-9_]+)`),
	"Instagram": regexp.MustCompile(`https?://(?:www\.)?instagram\.com/([a-zA-Z0-9_.]+)`),
	"GitHub":    regexp.MustCompile(`https?://(?:www\.)?github\.com/([a-zA-Z0-9\-]+)`),
	"Facebook":  regexp.MustCompile(`https?://(?:www\.)?facebook\.com/([a-zA-Z0-9.]+)`),
}

// Dead account indicators per platform
var deadIndicators = map[string][]string{
	"LinkedIn":  {"This page doesn't exist", "Page not found", "profile isn't available"},
	"Twitter/X": {"This account doesn't exist", "account suspended", "Hmm...this page doesn't exist"},
	"Instagram": {"Sorry, this page isn't available", "Page Not Found"},
	"GitHub":    {"Not Found", "404"},
	"Facebook":  {"This page isn't available", "content isn't available"},
}

type SocialLink struct {
	Platform string
	URL      string
	Username string
	FoundOn  string
}

type Result struct {
	Link   SocialLink
	Status string // "DEAD", "ALIVE", "UNKNOWN"
	Code   int
}

var (
	client *http.Client
	mu     sync.Mutex
)

func init() {
	client = &http.Client{
		Timeout: 15 * time.Second,
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
		},
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			return nil // follow redirects
		},
	}
}

func banner() {
	fmt.Printf(`%s%s
██████╗ ██╗     ██╗  ██╗    ███████╗ ██████╗ █████╗ ███╗   ██╗
██╔══██╗██║     ██║  ██║    ██╔════╝██╔════╝██╔══██╗████╗  ██║
██████╔╝██║     ███████║    ███████╗██║     ███████║██╔██╗ ██║
██╔══██╗██║     ██╔══██║    ╚════██║██║     ██╔══██║██║╚██╗██║
██████╔╝███████╗██║  ██║    ███████║╚██████╗██║  ██║██║ ╚████║
╚═════╝ ╚══════╝╚═╝  ╚═╝    ╚══════╝ ╚═════╝╚═╝  ╚═╝╚═╝  ╚═══╝
%s`, colorCyan, colorBold, colorReset)
	fmt.Printf("%s  Broken Link Hijacking - Social Media Scanner%s\n", colorYellow, colorReset)
	fmt.Printf("%s  by: kamu sendiri :)%s\n\n", colorReset, colorReset)
}

func fetchPage(targetURL string) (string, int, error) {
	req, err := http.NewRequest("GET", targetURL, nil)
	if err != nil {
		return "", 0, err
	}
	req.Header.Set("User-Agent", "Mozilla/5.0 (compatible; BLHScanner/1.0)")

	resp, err := client.Do(req)
	if err != nil {
		return "", 0, err
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(io.LimitReader(resp.Body, 1024*1024)) // max 1MB
	if err != nil {
		return "", resp.StatusCode, err
	}

	return string(body), resp.StatusCode, nil
}

func extractSocialLinks(body, pageURL string) []SocialLink {
	var links []SocialLink
	seen := map[string]bool{}

	for platform, pattern := range socialPatterns {
		matches := pattern.FindAllStringSubmatch(body, -1)
		for _, match := range matches {
			if len(match) < 2 {
				continue
			}
			fullURL := match[0]
			username := match[1]

			// Skip generic/reserved usernames
			if isGeneric(username) {
				continue
			}

			key := platform + ":" + username
			if seen[key] {
				continue
			}
			seen[key] = true

			links = append(links, SocialLink{
				Platform: platform,
				URL:      fullURL,
				Username: username,
				FoundOn:  pageURL,
			})
		}
	}

	return links
}

func isGeneric(username string) bool {
	genericNames := []string{
		"share", "sharer", "intent", "home", "login", "signup",
		"company", "about", "help", "legal", "privacy", "terms",
		"in", "pub", "dir", "search", "jobs", "learning",
	}
	lower := strings.ToLower(username)
	for _, g := range genericNames {
		if lower == g {
			return true
		}
	}
	return len(username) < 3
}

func checkSocialAccount(link SocialLink) Result {
	body, code, err := fetchPage(link.URL)
	if err != nil {
		return Result{Link: link, Status: "UNKNOWN", Code: 0}
	}

	// Hard 404
	if code == 404 {
		return Result{Link: link, Status: "DEAD", Code: code}
	}

	// Check body for dead indicators
	indicators := deadIndicators[link.Platform]
	bodyLower := strings.ToLower(body)
	for _, indicator := range indicators {
		if strings.Contains(bodyLower, strings.ToLower(indicator)) {
			return Result{Link: link, Status: "DEAD", Code: code}
		}
	}

	return Result{Link: link, Status: "ALIVE", Code: code}
}

func crawlPage(targetURL string) []SocialLink {
	fmt.Printf("%s[*] Crawling:%s %s\n", colorCyan, colorReset, targetURL)

	body, code, err := fetchPage(targetURL)
	if err != nil {
		fmt.Printf("%s[!] Error fetching %s: %v%s\n", colorRed, targetURL, err, colorReset)
		return nil
	}

	if code != 200 {
		fmt.Printf("%s[!] Got status %d for %s%s\n", colorYellow, code, targetURL, colorReset)
	}

	links := extractSocialLinks(body, targetURL)
	fmt.Printf("%s[+] Found %d social links on %s%s\n", colorGreen, len(links), targetURL, colorReset)
	return links
}

func crawlSubpages(baseURL string, depth int) []SocialLink {
	var allLinks []SocialLink
	visited := map[string]bool{baseURL: true}

	// Extract subpage links from homepage
	body, _, err := fetchPage(baseURL)
	if err != nil {
		return crawlPage(baseURL)
	}

	allLinks = append(allLinks, extractSocialLinks(body, baseURL)...)

	if depth <= 0 {
		return allLinks
	}

	// Find internal links
	hrefPattern := regexp.MustCompile(`href=["']([^"'#?]+)["']`)
	matches := hrefPattern.FindAllStringSubmatch(body, -1)

	base, _ := url.Parse(baseURL)
	checked := 0

	for _, match := range matches {
		if checked >= 20 { // limit subpage crawl
			break
		}
		href := match[1]
		if strings.HasPrefix(href, "http") {
			u, err := url.Parse(href)
			if err != nil || u.Host != base.Host {
				continue
			}
		} else if strings.HasPrefix(href, "/") {
			href = base.Scheme + "://" + base.Host + href
		} else {
			continue
		}

		if visited[href] {
			continue
		}
		visited[href] = true
		checked++

		subLinks := crawlPage(href)
		allLinks = append(allLinks, subLinks...)
		time.Sleep(300 * time.Millisecond)
	}

	return allLinks
}

func dedup(links []SocialLink) []SocialLink {
	seen := map[string]bool{}
	var result []SocialLink
	for _, l := range links {
		key := l.Platform + ":" + l.Username
		if !seen[key] {
			seen[key] = true
			result = append(result, l)
		}
	}
	return result
}

func main() {
	banner()

	target := flag.String("u", "", "Target URL (e.g. https://nasa.gov)")
	depth := flag.Int("d", 1, "Crawl depth for subpages (0=homepage only)")
	workers := flag.Int("w", 5, "Number of concurrent workers")
	output := flag.String("o", "", "Output file (optional)")
	flag.Parse()

	if *target == "" {
		fmt.Printf("%s[!] Usage: blh-scan -u https://target.com [-d 1] [-w 5] [-o output.txt]%s\n", colorRed, colorReset)
		os.Exit(1)
	}

	// Normalize URL
	if !strings.HasPrefix(*target, "http") {
		*target = "https://" + *target
	}

	fmt.Printf("%s[*] Target   :%s %s\n", colorCyan, colorReset, *target)
	fmt.Printf("%s[*] Depth    :%s %d\n", colorCyan, colorReset, *depth)
	fmt.Printf("%s[*] Workers  :%s %d\n\n", colorCyan, colorReset, *workers)

	// Step 1: Crawl
	fmt.Printf("%s=== PHASE 1: Crawling ===%s\n", colorBold, colorReset)
	allLinks := crawlSubpages(*target, *depth)
	allLinks = dedup(allLinks)

	if len(allLinks) == 0 {
		fmt.Printf("%s[!] No social media links found.%s\n", colorYellow, colorReset)
		os.Exit(0)
	}

	fmt.Printf("\n%s[+] Total unique social links found: %d%s\n\n", colorGreen, len(allLinks), colorReset)

	// Step 2: Check each account
	fmt.Printf("%s=== PHASE 2: Checking Accounts ===%s\n", colorBold, colorReset)

	jobs := make(chan SocialLink, len(allLinks))
	results := make(chan Result, len(allLinks))

	// Workers
	var wg sync.WaitGroup
	for i := 0; i < *workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for link := range jobs {
				fmt.Printf("  %s[~]%s Checking @%s on %s...\n", colorYellow, colorReset, link.Username, link.Platform)
				result := checkSocialAccount(link)
				results <- result
				time.Sleep(500 * time.Millisecond)
			}
		}()
	}

	for _, link := range allLinks {
		jobs <- link
	}
	close(jobs)

	wg.Wait()
	close(results)

	// Step 3: Report
	fmt.Printf("\n%s=== PHASE 3: Results ===%s\n\n", colorBold, colorReset)

	var deadLinks []Result
	var aliveLinks []Result
	var unknownLinks []Result

	for r := range results {
		switch r.Status {
		case "DEAD":
			deadLinks = append(deadLinks, r)
		case "ALIVE":
			aliveLinks = append(aliveLinks, r)
		default:
			unknownLinks = append(unknownLinks, r)
		}
	}

	var sb strings.Builder

	// Dead accounts (most important)
	if len(deadLinks) > 0 {
		header := fmt.Sprintf("🔴 DEAD / HIJACKABLE ACCOUNTS (%d found)\n", len(deadLinks))
		fmt.Printf("%s%s%s%s", colorRed, colorBold, header, colorReset)
		sb.WriteString(header)
		for _, r := range deadLinks {
			line := fmt.Sprintf("  [DEAD] %-12s | @%-30s | %s\n        Found on: %s\n",
				r.Link.Platform, r.Link.Username, r.Link.URL, r.Link.FoundOn)
			fmt.Printf("%s%s%s", colorRed, line, colorReset)
			sb.WriteString(line)
		}
	} else {
		fmt.Printf("%s[*] No dead/hijackable accounts found.%s\n", colorYellow, colorReset)
	}

	fmt.Println()

	// Alive accounts
	if len(aliveLinks) > 0 {
		header := fmt.Sprintf("🟢 ALIVE ACCOUNTS (%d)\n", len(aliveLinks))
		fmt.Printf("%s%s%s%s", colorGreen, colorBold, header, colorReset)
		sb.WriteString(header)
		for _, r := range aliveLinks {
			line := fmt.Sprintf("  [ALIVE] %-11s | @%-30s | %s\n",
				r.Link.Platform, r.Link.Username, r.Link.URL)
			fmt.Printf("%s%s%s", colorGreen, line, colorReset)
			sb.WriteString(line)
		}
	}

	// Summary
	summary := fmt.Sprintf("\n[SUMMARY] Dead: %d | Alive: %d | Unknown: %d | Total: %d\n",
		len(deadLinks), len(aliveLinks), len(unknownLinks), len(allLinks))
	fmt.Printf("%s%s%s%s", colorBold, colorCyan, summary, colorReset)
	sb.WriteString(summary)

	// Save output
	if *output != "" {
		err := os.WriteFile(*output, []byte(sb.String()), 0644)
		if err != nil {
			fmt.Printf("%s[!] Failed to save output: %v%s\n", colorRed, err, colorReset)
		} else {
			fmt.Printf("%s[+] Output saved to: %s%s\n", colorGreen, *output, colorReset)
		}
	}
}
