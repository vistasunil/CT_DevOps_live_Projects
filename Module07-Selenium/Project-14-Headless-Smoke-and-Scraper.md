# Project 14 — Headless Smoke Pack & Competitor Data Scraper (Windows)

| | |
|---|---|
| **Module** | 7 — Selenium |
| **Level** | L2 — Engineer |
| **Role you play** | SDET building the pipeline-ready test pack and a data-collection utility |
| **Platform** | **Windows 10/11** (matching the Module 7 hands-on) |
| **Scope** | `3-Headless-test.md`, `4-Selenium-webscrapping.md`, plus Project 13 foundations |

---

## 1. Business problem

Two requests landed in the same sprint:

1. **Release engineering** wants the smoke pack to run on a build agent that has **no display** —
   the current suite opens a visible browser and fails on the server.
2. **Product management** manually copies competitor pricing from a public website into a
   spreadsheet every Monday — four hours of copy-paste, with transcription errors.

Both are solved with the same skill: driving a browser headlessly and extracting data reliably.

## 2. Objectives

1. Convert the smoke pack to **headless** execution with a configurable toggle.
2. Make it CI-ready: exit codes, machine-readable results, fixed window size, no popups.
3. Build a **scraper** that extracts structured data and writes CSV/JSON.
4. Handle pagination, dynamic content and missing fields without crashing.
5. Document the ethical/legal checklist for scraping — this matters in a real job.

## 3. Scope

**In scope:** `ChromeOptions` (`--headless=new`, `--window-size`, `--no-sandbox`,
`--disable-dev-shm-usage`, `--disable-gpu`), `FirefoxOptions`, locators, explicit waits,
`JavascriptExecutor`, `findElements`, pagination loops, `TakesScreenshot`, file I/O,
TestNG parameters, Maven profiles.

**Out of scope:** Jenkins integration itself (Module 8), Selenium Grid, CAPTCHA bypass,
authenticated scraping of private data.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** QA-SOP-014 · **Title:** Headless test execution and web data extraction

### 4.1 Headless execution rules

| # | Rule | Reason |
|---|---|---|
| 1 | Use `--headless=new` (Chrome's modern headless) | The legacy `--headless` mode is deprecated |
| 2 | Always set `--window-size=1920,1080` | Headless defaults to a tiny viewport; responsive layouts hide elements |
| 3 | Headless is a **toggle**, not a fork of the code | One suite, two run modes |
| 4 | Capture a screenshot on failure even headless | It is the only way to see what happened |
| 5 | The build fails if the suite fails — never `-Dmaven.test.failure.ignore=true` on a gate | A green build with failing tests is worse than no tests |
| 6 | Pin the browser/driver on the agent | Reproducible results |

### 4.2 Web scraping checklist (do this **before** writing code)

| # | Check | Why |
|---|---|---|
| 1 | Read the site's **Terms of Service** | Scraping may be contractually prohibited |
| 2 | Check `/robots.txt` | Respect disallowed paths |
| 3 | Is there an official **API**? Use it instead | More stable, legal, cheaper |
| 4 | Only collect **public, non-personal** data | GDPR / privacy exposure |
| 5 | Rate-limit requests (≥1–2 s between pages) | Do not degrade someone else's service |
| 6 | Identify yourself honestly if a UA is required | Do not impersonate |
| 7 | Get written sign-off from your manager/legal for commercial use | Protects you personally |

> If any of checks 1–4 fail, **stop and escalate**. "The script worked" is not a defence.

### 4.3 Scraper reliability rules
1. Never assume a field exists — missing data returns `"N/A"`, it does not throw.
2. Log the count scraped per page and assert it is non-zero — a silent zero means the site
   changed and your spreadsheet is now wrong.
3. Write output atomically with a timestamped filename; never overwrite yesterday's data.
4. Re-run safe: the scraper must be idempotent and restartable.

---

## 5. Hands-on implementation (Windows)

### Step 1 — Driver factory with a headless toggle

`src/test/java/com/acme/qa/base/DriverFactory.java`:

```java
package com.acme.qa.base;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;

import java.time.Duration;

public final class DriverFactory {

    private DriverFactory() { }

    public static WebDriver create() {
        boolean headless = Boolean.parseBoolean(System.getProperty("headless", "false"));
        String browser   = System.getProperty("browser", "chrome").toLowerCase();

        WebDriver driver = switch (browser) {
            case "firefox" -> firefox(headless);
            default        -> chrome(headless);
        };

        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(5));
        driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(40));
        if (!headless) {
            driver.manage().window().maximize();
        }
        return driver;
    }

    private static WebDriver chrome(boolean headless) {
        ChromeOptions o = new ChromeOptions();
        if (headless) {
            o.addArguments("--headless=new");      // modern headless mode
            o.addArguments("--window-size=1920,1080");
            o.addArguments("--disable-gpu");
            o.addArguments("--no-sandbox");               // required on many CI agents
            o.addArguments("--disable-dev-shm-usage");    // avoids /dev/shm exhaustion
        }
        o.addArguments("--disable-notifications");
        o.addArguments("--disable-popup-blocking");
        o.addArguments("--remote-allow-origins=*");
        return new ChromeDriver(o);                // Selenium Manager resolves the driver
    }

    private static WebDriver firefox(boolean headless) {
        FirefoxOptions o = new FirefoxOptions();
        if (headless) {
            o.addArguments("-headless");
            o.addArguments("--width=1920");
            o.addArguments("--height=1080");
        }
        return new FirefoxDriver(o);
    }
}
```

Update `BaseTest` to use it:

```java
@BeforeMethod(alwaysRun = true)
public void setUp() {
    driver = DriverFactory.create();
    wait = new WebDriverWait(driver, Duration.ofSeconds(15));
    driver.get(BASE_URL);
}
```

### Step 2 — Prove headless works and the viewport is correct

`src/test/java/com/acme/qa/tests/HeadlessSanityTests.java`:

```java
package com.acme.qa.tests;

import com.acme.qa.base.BaseTest;
import org.openqa.selenium.Dimension;
import org.openqa.selenium.JavascriptExecutor;
import org.testng.Assert;
import org.testng.annotations.Test;

public class HeadlessSanityTests extends BaseTest {

    @Test(groups = {"smoke"}, description = "TC-20 Viewport is desktop-sized in headless mode")
    public void viewportIsDesktopSized() {
        Dimension size = driver.manage().window().getSize();
        System.out.println("Viewport: " + size.getWidth() + "x" + size.getHeight());
        Assert.assertTrue(size.getWidth() >= 1280,
                "Viewport too narrow (" + size.getWidth() + ") - responsive layout may hide elements");
    }

    @Test(groups = {"smoke"}, description = "TC-21 Page fully loads (document.readyState)")
    public void pageReachesReadyState() {
        String state = (String) ((JavascriptExecutor) driver)
                .executeScript("return document.readyState");
        Assert.assertEquals(state, "complete", "Page did not reach readyState=complete");
    }

    @Test(groups = {"smoke"}, description = "TC-22 Page title is present")
    public void titleIsPresent() {
        Assert.assertFalse(driver.getTitle().isBlank(), "Page title is empty");
    }
}
```

### Step 3 — Run headed vs headless

```bat
:: Headed (local debugging)
mvn clean test -Dheadless=false

:: Headless (what the build agent will run)
mvn clean test -Dheadless=true

:: Headless Firefox
mvn clean test -Dheadless=true -Dbrowser=firefox

:: Smoke group only, headless - the future pipeline gate
mvn clean test -Dheadless=true -Dgroups=smoke
```

Confirm the exit code is what a pipeline will read:

```bat
mvn clean test -Dheadless=true
echo Exit code: %ERRORLEVEL%
:: 0 = all passed, non-zero = failures -> the pipeline must fail
```

### Step 4 — A pipeline-ready smoke profile in `pom.xml`

```xml
<profiles>
  <profile>
    <id>ci</id>
    <properties>
      <headless>true</headless>
    </properties>
    <build>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-surefire-plugin</artifactId>
          <version>3.2.5</version>
          <configuration>
            <suiteXmlFiles><suiteXmlFile>testng-smoke.xml</suiteXmlFile></suiteXmlFiles>
            <systemPropertyVariables>
              <headless>true</headless>
            </systemPropertyVariables>
            <!-- do NOT ignore failures: the gate must break the build -->
            <testFailureIgnore>false</testFailureIgnore>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

```bat
mvn clean test -Pci
```

### Step 5 — The scraper

`src/test/java/com/acme/qa/scraper/ProductScraper.java`:

```java
package com.acme.qa.scraper;

import com.acme.qa.base.DriverFactory;
import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;

import java.nio.file.*;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.*;

/**
 * Extracts public book listings from the sandbox site books.toscrape.com,
 * which exists explicitly for scraping practice.
 *
 * BEFORE running against any other site, complete the checklist in QA-SOP-014 section 4.2.
 */
public class ProductScraper {

    private static final String START_URL = "https://books.toscrape.com/catalogue/page-1.html";
    private static final int    MAX_PAGES = 5;
    private static final long   POLITE_DELAY_MS = 1500;   // rate limit - be a good citizen

    record Product(String title, String price, String availability, String rating, String url) {
        String toCsvRow() {
            return String.join(",",
                    quote(title), quote(price), quote(availability), quote(rating), quote(url));
        }
        private static String quote(String s) {
            return "\"" + (s == null ? "N/A" : s.replace("\"", "\"\"")) + "\"";
        }
    }

    public static void main(String[] args) throws Exception {
        System.setProperty("headless", "true");      // scraping always runs headless
        WebDriver driver = DriverFactory.create();
        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(20));
        List<Product> all = new ArrayList<>();

        try {
            String url = START_URL;
            for (int page = 1; page <= MAX_PAGES && url != null; page++) {
                System.out.println("Scraping page " + page + ": " + url);
                driver.get(url);

                wait.until(ExpectedConditions.presenceOfElementLocated(
                        By.cssSelector("article.product_pod")));

                List<WebElement> cards = driver.findElements(By.cssSelector("article.product_pod"));
                System.out.println("  found " + cards.size() + " items");

                if (cards.isEmpty()) {
                    throw new IllegalStateException(
                        "0 items found - the page structure probably changed. Aborting so we "
                      + "do not silently publish an empty dataset.");
                }

                for (WebElement card : cards) {
                    all.add(new Product(
                            text(card, By.cssSelector("h3 a"), "title"),
                            text(card, By.cssSelector(".price_color"), null),
                            text(card, By.cssSelector(".instock.availability"), null),
                            attr(card, By.cssSelector("p.star-rating"), "class"),
                            attr(card, By.cssSelector("h3 a"), "href")
                    ));
                }

                url = nextPageUrl(driver);
                Thread.sleep(POLITE_DELAY_MS);   // deliberate rate limit, not a test wait
            }

            writeCsv(all);
            writeJson(all);
            System.out.println("Scraped " + all.size() + " products across pages.");

        } finally {
            driver.quit();
        }
    }

    /** Never throws on a missing field - returns N/A so one odd card cannot kill the run. */
    private static String text(WebElement scope, By by, String attrFallback) {
        try {
            WebElement e = scope.findElement(by);
            String t = e.getText().trim();
            if (t.isEmpty() && attrFallback != null) t = e.getAttribute(attrFallback);
            return t.isEmpty() ? "N/A" : t;
        } catch (NoSuchElementException e) {
            return "N/A";
        }
    }

    private static String attr(WebElement scope, By by, String attribute) {
        try {
            String v = scope.findElement(by).getAttribute(attribute);
            return (v == null || v.isBlank()) ? "N/A" : v.replace("star-rating", "").trim();
        } catch (NoSuchElementException e) {
            return "N/A";
        }
    }

    private static String nextPageUrl(WebDriver driver) {
        try {
            return driver.findElement(By.cssSelector("li.next a")).getAttribute("href");
        } catch (NoSuchElementException e) {
            return null;       // last page
        }
    }

    private static void writeCsv(List<Product> products) throws Exception {
        Path out = outputPath("csv");
        StringBuilder sb = new StringBuilder("title,price,availability,rating,url\n");
        products.forEach(p -> sb.append(p.toCsvRow()).append("\n"));
        Files.writeString(out, sb.toString());
        System.out.println("CSV written: " + out.toAbsolutePath());
    }

    private static void writeJson(List<Product> products) throws Exception {
        Path out = outputPath("json");
        StringBuilder sb = new StringBuilder("[\n");
        for (int i = 0; i < products.size(); i++) {
            Product p = products.get(i);
            sb.append(String.format(
                "  {\"title\":%s,\"price\":%s,\"availability\":%s,\"rating\":%s,\"url\":%s}%s%n",
                j(p.title()), j(p.price()), j(p.availability()), j(p.rating()), j(p.url()),
                i < products.size() - 1 ? "," : ""));
        }
        sb.append("]\n");
        Files.writeString(out, sb.toString());
        System.out.println("JSON written: " + out.toAbsolutePath());
    }

    private static String j(String s) {
        return "\"" + (s == null ? "N/A" : s.replace("\\", "\\\\").replace("\"", "\\\"")) + "\"";
    }

    private static Path outputPath(String ext) throws Exception {
        Files.createDirectories(Paths.get("target/scrape"));
        String stamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss"));
        return Paths.get("target/scrape", "products-" + stamp + "." + ext);
    }
}
```

### Step 6 — Run the scraper

```bat
mvn -q compile
mvn -q exec:java -Dexec.mainClass="com.acme.qa.scraper.ProductScraper" -Dexec.classpathScope=test

:: or from the IDE: run ProductScraper.main()

dir target\scrape
type target\scrape\products-*.csv | more
```

Add the exec plugin if needed:

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>exec-maven-plugin</artifactId>
  <version>3.1.1</version>
</plugin>
```

### Step 7 — A test that guards the scraper

`src/test/java/com/acme/qa/tests/ScraperGuardTests.java`:

```java
package com.acme.qa.tests;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.support.ui.*;
import com.acme.qa.base.DriverFactory;
import org.testng.Assert;
import org.testng.annotations.*;

import java.time.Duration;

/** If the source site changes its markup, this fails BEFORE the scraper ships bad data. */
public class ScraperGuardTests {

    private WebDriver driver;

    @BeforeClass(alwaysRun = true)
    public void setUp() {
        System.setProperty("headless", "true");
        driver = DriverFactory.create();
    }

    @Test(groups = {"scraper"}, description = "TC-30 Source markup contract still holds")
    public void sourceMarkupContractHolds() {
        driver.get("https://books.toscrape.com/catalogue/page-1.html");
        new WebDriverWait(driver, Duration.ofSeconds(20))
                .until(ExpectedConditions.presenceOfElementLocated(
                        By.cssSelector("article.product_pod")));

        Assert.assertTrue(driver.findElements(By.cssSelector("article.product_pod")).size() > 0,
                "No product cards - selector contract broken");
        Assert.assertTrue(driver.findElements(By.cssSelector(".price_color")).size() > 0,
                "No price elements - selector contract broken");
        Assert.assertTrue(driver.findElements(By.cssSelector("li.next a")).size() > 0,
                "No pagination link - pagination contract broken");
    }

    @AfterClass(alwaysRun = true)
    public void tearDown() {
        if (driver != null) driver.quit();
    }
}
```

### Step 8 — Measure the win

```bat
:: Time the headless smoke run
powershell -Command "Measure-Command { mvn -q clean test -Pci }"

:: Time the scraper
powershell -Command "Measure-Command { mvn -q exec:java -Dexec.mainClass='com.acme.qa.scraper.ProductScraper' -Dexec.classpathScope=test }"
```

Record: manual effort before vs automated runtime now.

---

## 6. Acceptance criteria

- [ ] One codebase runs headed and headless via `-Dheadless=true/false`
- [ ] Headless viewport is ≥1280 px wide (asserted by a test)
- [ ] `--headless=new` used, not the deprecated flag
- [ ] Failing headless run still produces a screenshot
- [ ] `mvn test -Pci` returns a non-zero exit code when a test fails
- [ ] Scraper paginates, rate-limits, and writes timestamped CSV **and** JSON
- [ ] Missing fields produce `N/A`, never an exception
- [ ] Scraper aborts loudly if a page yields zero items
- [ ] Scraping checklist completed and documented for the target site
- [ ] `driver.quit()` in a `finally` block — no orphaned processes

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| Works headed, fails headless | Tiny default viewport hides elements | `--window-size=1920,1080` |
| `DevToolsActivePort file doesn't exist` | Sandbox/shared memory restrictions on the agent | `--no-sandbox --disable-dev-shm-usage` |
| `unknown error: cannot connect to chrome` | Browser/driver mismatch or no browser installed | Install Chrome on the agent; let Selenium Manager resolve the driver |
| Scraper returns 0 rows but exits 0 | No zero-result guard | Throw/fail on empty page (implemented in Step 5) |
| Scraper blocked / 429 | Too fast, no rate limit | Increase `POLITE_DELAY_MS`; respect `robots.txt` |
| Intermittent `NoSuchElement` while scraping | Lazy-loaded content | Explicit wait on the container before reading children |
| CSV broken in Excel | Commas/quotes inside fields | Quote and escape (`""`) — implemented in `Product.toCsvRow()` |
| Build "passes" with failing tests | `testFailureIgnore=true` | Set it to `false` on any gating job |
| Orphan `chrome.exe` on the agent | Missing `quit()` on an exception path | `try/finally` around the whole run |

---

## 8. Deliverables

1. Repo additions: `DriverFactory`, headless sanity tests, `ProductScraper`, `ScraperGuardTests`, `ci` Maven profile, `testng-smoke.xml`.
2. `QA-SOP-014.md` — headless rules + the scraping legal/ethical checklist.
3. Sample `products-<timestamp>.csv` and `.json` outputs.
4. Evidence: headed vs headless run logs, exit-code demonstration, a failing-run screenshot.
5. A short note on the target site's ToS/robots.txt findings and the decision made.

---

## 9. Interview talking points

* *"Why does a test pass locally but fail on the build agent?"* — Usually viewport/headless
  differences, missing browser, or sandbox restrictions; fix with explicit window size and the
  right Chrome flags, not by disabling the test.
* *"What is `--headless=new`?"* — Chrome's current headless implementation; the old mode is
  deprecated and behaves differently from real Chrome.
* *"How do you debug a headless failure?"* — Screenshot + page source on failure, browser logs,
  and the ability to re-run the same test headed with one flag.
* *"Is scraping legal?"* — It depends: Terms of Service, `robots.txt`, personal data, and
  commercial use. I check those first, prefer an official API, rate-limit, and get sign-off.
* *"How do you stop a scraper silently producing bad data?"* — Guard tests on the markup
  contract, a non-zero-row assertion, timestamped output files, and logging counts per page.
* *"How would this plug into CI?"* — `mvn test -Pci` as a build step; non-zero exit fails the
  job; publish the TestNG/Surefire report and archive screenshots as artefacts (Module 8).

**CV bullet:** *Converted the Selenium regression pack to pipeline-ready headless execution
(Chrome `--headless=new`, fixed viewport, failure screenshots, enforced exit codes) and delivered
a rate-limited, fault-tolerant web data extractor producing timestamped CSV/JSON, replacing a
weekly four-hour manual data-collection task.*
