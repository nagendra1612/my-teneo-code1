# my-teneo-code1
for my airdrop buddy v2
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"net/url"
	"os"
	"sort"
	"strings"
	"time"

	"github.com/TeneoProtocolAI/teneo-agent-sdk/pkg/agent"
	"github.com/joho/godotenv"
	"golang.org/x/time/rate"
)

// ============================================================================
// 1. CONFIGURATION & GLOBALS
// ============================================================================

var (
	// Rate Limiter: 10 requests per minute (Safe for CoinGecko Free Tier)
	coingeckoLimiter = rate.NewLimiter(rate.Every(time.Minute/10), 1)
	httpClient       = &http.Client{Timeout: 15 * time.Second} // Increased timeout for stability
)

// ============================================================================
// 2. REASONING ENGINE (The "Why" Logic)
// ============================================================================

type CommandLogic struct {
	Description   string
	ReasoningSpec string
}

var CommandRegistry = map[string]CommandLogic{
	// Global Commands
	"spotlight": {
		Description:   "Top 3 Trending 'Rising Stars'",
		ReasoningSpec: "Show price, change, and explain trend via sector narrative (AI, Meme, RWA) and momentum.",
	},
	"high_growth": {
		Description:   "Top Gainers Radar",
		ReasoningSpec: "Explain growth: viral breakout, sector rotation, or whale accumulation.",
	},
	"new_airdrop": {
		Description:   "Airdrop Watchlist",
		ReasoningSpec: "List projects with points/testnets. Explain why an airdrop is likely.",
	},
	"track_launches": {
		Description:   "Launch Alpha Checklist",
		ReasoningSpec: "Sources for new coins and what specific alpha they provide.",
	},
	"incentive_watch": {
		Description:   "Active Rewards Programs",
		ReasoningSpec: "Current programs and their specific triggers (Volume, LPing, Bridging).",
	},
	"help": {
		Description:   "Command Guide",
		ReasoningSpec: "List commands with usage context.",
	},
	// Coin Specific
	"project_info": {
		Description:   "Project Fundamentals",
		ReasoningSpec: "Interpret Market Cap & ATH distance to explain risk/reward ratio.",
	},
	"simple_updates": {
		Description:   "Price Action & Momentum",
		ReasoningSpec: "Interpret 24h/7d moves: Breakout, Consolidation, or Correction.",
	},
	"funding_status": {
		Description:   "VC Backing Analysis",
		ReasoningSpec: "Check description for investors. Explain VC vs. Community/Fair Launch dynamics.",
	},
	"score_card": {
		Description:   "Developer & Liquidity Grade",
		ReasoningSpec: "Grade A-C based on activity/liquidity. Explain entry/exit ease.",
	},
	"risk_simple": {
		Description:   "Risk Classification",
		ReasoningSpec: "Classify Low/Med/High based on Mcap tiers.",
	},
	"audit_status": {
		Description:   "Security Check",
		ReasoningSpec: "Remind that price != security. Prompt to check audits.",
	},
	"red_flag": {
		Description:   "Safety Check",
		ReasoningSpec: "Warn if liquidity is critically low.",
	},
}

// ============================================================================
// 3. API STRUCTS
// ============================================================================

type CoinMarketItem struct {
	ID             string  `json:"id"`
	Symbol         string  `json:"symbol"`
	Name           string  `json:"name"`
	CurrentPrice   float64 `json:"current_price"`
	MarketCap      float64 `json:"market_cap"`
	MarketCapRank  int     `json:"market_cap_rank"`
	TotalVolume    float64 `json:"total_volume"`
	PriceChange24h float64 `json:"price_change_percentage_24h"`
	ATH            float64 `json:"ath"`
}

type DetailedCoinData struct {
	ID             string  `json:"id"`
	Symbol         string  `json:"symbol"`
	Name           string  `json:"name"`
	MarketCapRank  int     `json:"market_cap_rank"`
	LiquidityScore float64 `json:"liquidity_score"`
	DeveloperScore float64 `json:"developer_score"`
	Description    struct {
		En string `json:"en"`
	} `json:"description"`
	Links struct {
		Homepage []string `json:"homepage"`
	} `json:"links"`
	MarketData struct {
		CurrentPrice struct {
			Usd float64 `json:"usd"`
		} `json:"current_price"`
		MarketCap struct {
			Usd float64 `json:"usd"`
		} `json:"market_cap"`
		PriceChange24h float64 `json:"price_change_percentage_24h"`
		PriceChange7d  float64 `json:"price_change_percentage_7d"`
		ATH            struct {
			Usd float64 `json:"usd"`
		} `json:"ath"`
	} `json:"market_data"`
	DeveloperData struct {
		Stars int `json:"stars"`
		Forks int `json:"forks"`
	} `json:"developer_data"`
}

type SearchResponse struct {
	Coins []struct {
		ID            string `json:"id"`
		Symbol        string `json:"symbol"`
		MarketCapRank *int   `json:"market_cap_rank"`
	} `json:"coins"`
}

type TrendingResponse struct {
	Coins []struct {
		Item struct {
			ID            string `json:"id"`
			Name          string `json:"name"`
			Symbol        string `json:"symbol"`
			MarketCapRank int    `json:"market_cap_rank"`
			Data          struct {
				PriceChangePercentage24h map[string]float64 `json:"price_change_percentage_24h"`
			} `json:"data"`
		} `json:"item"`
	} `json:"coins"`
}

// ============================================================================
// 4. NETWORK HELPERS
// ============================================================================

func doRequest(url string) (*http.Response, error) {
	if err := coingeckoLimiter.Wait(context.Background()); err != nil {
		return nil, err
	}
	req, _ := http.NewRequest("GET", url, nil)
	req.Header.Set("User-Agent", "Mozilla/5.0 (TeneoAgent/2.0)")
	return httpClient.Do(req)
}

// resolveCoinID matches user input (e.g., "solana", "BTC") to a valid CoinGecko ID
func resolveCoinID(query string) (string, error) {
	query = strings.ToLower(strings.TrimSpace(query))
	
	// 1. Search API
	searchURL := fmt.Sprintf("https://api.coingecko.com/api/v3/search?query=%s", url.QueryEscape(query))
	resp, err := doRequest(searchURL)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()

	var searchRes SearchResponse
	if err := json.NewDecoder(resp.Body).Decode(&searchRes); err != nil {
		return "", err
	}

	if len(searchRes.Coins) == 0 {
		return "", fmt.Errorf("not found")
	}

	// 2. Exact Match Strategy
	for _, coin := range searchRes.Coins {
		if strings.EqualFold(coin.ID, query) || strings.EqualFold(coin.Symbol, query) {
			return coin.ID, nil
		}
	}

	// 3. Fallback: Return the highest ranked result
	return searchRes.Coins[0].ID, nil
}

func fetchCoinDetails(id string) (*DetailedCoinData, error) {
	url := fmt.Sprintf("https://api.coingecko.com/api/v3/coins/%s?localization=false&tickers=false&market_data=true&community_data=false&developer_data=true", id)
	resp, err := doRequest(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var data DetailedCoinData
	if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
		return nil, err
	}
	return &data, nil
}

func fetchMarketBatch(ids []string) ([]CoinMarketItem, error) {
	idStr := strings.Join(ids, ",")
	url := fmt.Sprintf("https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&ids=%s&order=market_cap_desc&sparkline=false", idStr)
	resp, err := doRequest(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var data []CoinMarketItem
	if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
		return nil, err
	}
	return data, nil
}

// ============================================================================
// 5. COMMAND HANDLERS
// ============================================================================

func handleGlobalCommands(cmd string) string {
	var sb strings.Builder
	
	// Check registry for description
	if val, ok := CommandRegistry[cmd]; ok {
		sb.WriteString(fmt.Sprintf("🤖 **%s**\n_%s_\n\n", strings.ToUpper(cmd), val.ReasoningSpec))
	} else {
		// Fallback header
		sb.WriteString(fmt.Sprintf("🤖 **%s**\n\n", strings.ToUpper(cmd)))
	}

	switch cmd {
	case "spotlight":
		// Fetch Trending
		url := "https://api.coingecko.com/api/v3/search/trending"
		resp, err := doRequest(url)
		if err == nil {
			defer resp.Body.Close()
			var trend TrendingResponse
			json.NewDecoder(resp.Body).Decode(&trend)
			
			// Process top 3
			var ids []string
			for i, c := range trend.Coins {
				if i < 3 { ids = append(ids, c.Item.ID) }
			}
			
			if len(ids) > 0 {
				marketData, _ := fetchMarketBatch(ids)
				for _, coin := range marketData {
					sb.WriteString(fmt.Sprintf("🔥 **%s** ($%s): $%.4f\n", coin.Name, strings.ToUpper(coin.Symbol), coin.CurrentPrice))
					sb.WriteString(fmt.Sprintf("   Change: %.2f%% | Market Cap Rank: #%d\n\n", coin.PriceChange24h, coin.MarketCapRank))
				}
			} else {
				sb.WriteString("⚠️ Could not retrieve trending data right now.")
			}
		}

	case "high_growth":
		// Fetch Markets
		url := "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=100&page=1&sparkline=false"
		resp, err := doRequest(url)
		if err == nil {
			defer resp.Body.Close()
			var allCoins []CoinMarketItem
			json.NewDecoder(resp.Body).Decode(&allCoins)
			
			// Sort by Gainers
			sort.Slice(allCoins, func(i, j int) bool {
				return allCoins[i].PriceChange24h > allCoins[j].PriceChange24h
			})
			
			for i := 0; i < 3 && i < len(allCoins); i++ {
				c := allCoins[i]
				sb.WriteString(fmt.Sprintf("🚀 **%s** (+%.2f%%)\n", c.Name, c.PriceChange24h))
				sb.WriteString(fmt.Sprintf("   MCap: $%.0fM\n\n", c.MarketCap/1e6))
			}
		}

	case "new_airdrop":
		sb.WriteString("1. **Monad**: EVM L1. Why: Massive funding usually means community rewards.\n")
		sb.WriteString("2. **Berachain**: L1. Why: Active Testnet (Artio) often precedes mainnet drops.\n")
		sb.WriteString("3. **Hyperliquid**: DEX. Why: Points program is explicitly active.\n")

	case "track_launches":
		sb.WriteString("✅ **Launch Source Checklist**\n")
		sb.WriteString("- **CoinGecko New**: High risk, verify contracts.\n")
		sb.WriteString("- **DefiLlama**: Check 'Airdrops' tab for protocol incentives.\n")
		sb.WriteString("- **Twitter/X**: Search ticker cashtags ($SYMBOL) for sentiment.\n")

	case "incentive_watch":
		sb.WriteString("🎁 **Active Programs**\n")
		sb.WriteString("- **Blast**: Bridging ETH yields points.\n")
		sb.WriteString("- **Scroll**: Marks program for dApp usage.\n")
	
	case "help":
		sb.WriteString("📚 **Available Commands:**\n")
		for k, v := range CommandRegistry {
			sb.WriteString(fmt.Sprintf("- `/%s`: %s\n", k, v.Description))
		}
	}
	return sb.String()
}

func handleCoinCommands(cmd string, coinID string) string {
	data, err := fetchCoinDetails(coinID)
	if err != nil {
		return fmt.Sprintf("❌ Error fetching data for ID: %s. (Rate limit or invalid ID)", coinID)
	}

	var sb strings.Builder
	sb.WriteString(fmt.Sprintf("🔍 **Analysis: %s ($%s)**\n", data.Name, strings.ToUpper(data.Symbol)))
	
	// Add Reasoning Header if available
	if val, ok := CommandRegistry[cmd]; ok {
		sb.WriteString(fmt.Sprintf("👉 _%s_\n\n", val.ReasoningSpec))
	}

	switch cmd {
	case "project_info":
		sb.WriteString(fmt.Sprintf("💰 **Price:** $%.4f\n", data.MarketData.CurrentPrice.Usd))
		sb.WriteString(fmt.Sprintf("🏆 **Rank:** #%d\n", data.MarketCapRank))
		sb.WriteString(fmt.Sprintf("🧢 **Market Cap:** $%.0fM\n", data.MarketData.MarketCap.Usd/1e6))
		sb.WriteString(fmt.Sprintf("🏔️ **ATH:** $%.2f\n", data.MarketData.ATH.Usd))

	case "simple_updates":
		change := data.MarketData.PriceChange24h
		emoji := "😐"
		if change > 5 { emoji = "🚀" } else if change < -5 { emoji = "📉" }
		
		sb.WriteString(fmt.Sprintf("Price Action (24h): %.2f%% %s\n", change, emoji))
		sb.WriteString(fmt.Sprintf("Price Action (7d):  %.2f%%\n", data.MarketData.PriceChange7d))

	case "score_card":
		dev := data.DeveloperData.Stars + data.DeveloperData.Forks
		liq := data.LiquidityScore
		grade := "C"
		if dev > 200 && liq > 50 { grade = "A" } else if dev > 50 || liq > 30 { grade = "B" }
		
		sb.WriteString(fmt.Sprintf("📝 **Grade: %s**\n", grade))
		sb.WriteString(fmt.Sprintf("   Dev Activity: %d interactions\n", dev))
		sb.WriteString(fmt.Sprintf("   Liquidity Score: %.1f/100\n", liq))

	case "risk_simple":
		risk := "HIGH"
		if data.MarketData.MarketCap.Usd > 1e9 { risk = "LOW (Large Cap)" } else if data.MarketData.MarketCap.Usd > 1e8 { risk = "MEDIUM (Mid Cap)" }
		sb.WriteString(fmt.Sprintf("🛡️ **Risk Level: %s**\n", risk))
		sb.WriteString("   Reasoning: Market cap tier determines general volatility exposure.")

	case "funding_status":
		desc := strings.ToLower(data.Description.En)
		if strings.Contains(desc, "capital") || strings.Contains(desc, "venture") || strings.Contains(desc, "backed") {
			sb.WriteString("💰 **VC Backing Detected.**\n   Action: Research vesting schedules and unlock dates.")
		} else {
			sb.WriteString("🌱 **Likely Fair/Community Launch.**\n   Action: Watch for high volatility; no VC unlock pressure.")
		}

	case "audit_status":
		sb.WriteString(fmt.Sprintf("🌐 **Homepage:** %s\n", safeLink(data.Links.Homepage)))
		sb.WriteString("⚠️ **Security:** Check homepage for 'Audits' footer. Price != Safety.")

	case "red_flag":
		if data.LiquidityScore < 10 {
			sb.WriteString("🚩 **RED FLAG:** Extremely Low Liquidity. High rug risk.")
		} else {
			sb.WriteString("✅ **Sanity Check:** No critical liquidity flags detected.")
		}
	}

	return sb.String()
}

// ============================================================================
// 6. MAIN AGENT LOGIC
// ============================================================================

type Mr02Agent struct{}

func (a *Mr02Agent) ProcessTask(ctx context.Context, task string) (string, error) {
	// 1. INPUT CLEANING
	// Remove mentions like @mr-02
	if idx := strings.Index(task, "@"); idx != -1 {
		// If mention is at start "@mr-02 help", strip it
		parts := strings.Fields(task)
		cleanParts := []string{}
		for _, p := range parts {
			if !strings.HasPrefix(p, "@") {
				cleanParts = append(cleanParts, p)
			}
		}
		task = strings.Join(cleanParts, " ")
	}
	
	// Normalize
	task = strings.TrimSpace(task)
	parts := strings.Fields(task)
	if len(parts) == 0 {
		return "⚠️ Please provide a command. Type `/help`.", nil
	}

	// 2. PARSE COMMAND
	cmd := strings.ToLower(strings.TrimPrefix(parts[0], "/"))
	
	// 3. ROUTE GLOBAL COMMANDS
	if isGlobalCommand(cmd) {
		return handleGlobalCommands(cmd), nil
	}

	// 4. ROUTE COIN COMMANDS
	if len(parts) < 2 {
		return fmt.Sprintf("❌ The command `%s` requires a coin name.\nExample: `%s solana`", cmd, cmd), nil
	}
	
	// Join remaining parts to handle names like "baby doge coin"
	rawQuery := strings.Join(parts[1:], " ")
	
	// Resolve ID (The "Smart Search" Fix)
	coinID, err := resolveCoinID(rawQuery)
	if err != nil {
		return fmt.Sprintf("❌ Could not find coin matching '**%s**'. Try the full name or symbol.", rawQuery), nil
	}

	return handleCoinCommands(cmd, coinID), nil
}

// Helper to identify global vs specific commands
func isGlobalCommand(cmd string) bool {
	globals := []string{"spotlight", "high_growth", "new_airdrop", "track_launches", "incentive_watch", "help"}
	for _, g := range globals {
		if g == cmd { return true }
	}
	return false
}

func safeLink(links []string) string {
	if len(links) > 0 && links[0] != "" { return links[0] }
	return "N/A"
}

// ============================================================================
// 7. ENTRY POINT
// ============================================================================

func main() {
	_ = godotenv.Load()

	config := agent.DefaultConfig()
	config.Name = "Airdrop Buddy V2" // NAME UPDATED HERE
	config.Description = "Reasoning & Analysis Agent"
	config.Capabilities = []string{"market_analysis", "reasoning"}
	config.PrivateKey = os.Getenv("PRIVATE_KEY")
	config.NFTTokenID = os.Getenv("NFT_TOKEN_ID")
	config.OwnerAddress = os.Getenv("WALLET_ADDRESS")

	agentImpl, err := agent.NewEnhancedAgent(&agent.EnhancedAgentConfig{
		Config:       config,
		AgentHandler: &Mr02Agent{},
	})

	if err != nil {
		log.Fatal("💥 Failed to start agent:", err)
	}

	log.Println("🚀 Airdrop Buddy V2 is Online...")
	if err := agentImpl.Run(); err != nil {
		log.Fatal(err)
	}
}
