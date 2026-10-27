# Project 13 — Automated Regression Suite with Selenium + TestNG (Windows)

| | |
|---|---|
| **Module** | 7 — Selenium |
| **Level** | L2 — Engineer |
| **Role you play** | QA Automation / SDET engineer embedded in a delivery squad |
| **Platform** | **Windows 10/11** (matching the Module 7 hands-on) |
| **Scope** | `1-Selenium-setup.md`, `2-TestNg-annotations-selenium.md` |

---

## 1. Business problem

Before every release the QA team spends two days clicking through the same 40 test cases on the
web application. Regressions still reach production because testers get tired, skip steps and
record results inconsistently. There is no evidence trail of what was tested.

You must build an **automated regression suite** using Selenium WebDriver and TestNG that runs
the critical user journeys, produces a report, and can be handed to the build pipeline later
(Module 8).

## 2. Objectives

1. Set up a Java 21 + Maven + Selenium 4 project on Windows.
2. Implement the **Page Object Model** so tests survive UI changes.
3. Use the full TestNG annotation lifecycle, groups, priorities, dependencies and data providers.
4. Produce a suite that runs via `mvn test` and emits a readable report with screenshots on failure.
5. Document the test design and the triage procedure for a failing test.

## 3. Scope

**In scope:** Selenium 4 WebDriver + Selenium Manager, locators (`By.id/name/cssSelector/xpath`),
waits (explicit/implicit), `WebElement` interactions, `Actions`, `Select`, alerts, windows/frames,
TestNG `@BeforeSuite/@BeforeClass/@BeforeMethod/@Test/@AfterMethod/@AfterClass/@AfterSuite`,
`priority`, `dependsOnMethods`, `groups`, `@DataProvider`, `Assert`, `testng.xml`, Maven Surefire.

**Out of scope:** headless/CI execution (Project 14), BDD/Cucumber, mobile, performance testing.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** QA-SOP-013 · **Title:** Build and operate the web regression suite

### 4.1 Test design rules

| # | Rule | Reason |
|---|---|---|
| 1 | **Page Object Model** — locators live in page classes only | A UI change is one file, not 40 tests |
| 2 | **No `Thread.sleep()`** — use `WebDriverWait` | Sleeps are slow and still flaky |
| 3 | Prefer `id` > `name` > `cssSelector` > `xpath` | Stability; avoid absolute XPath entirely |
| 4 | One assertion theme per test | A failure names the defect |
| 5 | Tests are independent and order-free | Any test can run alone |
| 6 | Test data from `@DataProvider`/properties, never hard-coded | Reuse and coverage |
| 7 | Screenshot + page source captured on failure | Triage without re-running |
| 8 | `driver.quit()` in `@AfterMethod`/`@AfterClass` | Leaked browsers exhaust the agent |

### 4.2 Failure triage procedure

| # | Step |
|---|---|
| 1 | Open the TestNG report; note the failing test and exception type |
| 2 | Look at the captured screenshot — is it a real defect or a locator problem? |
| 3 | `NoSuchElementException` / `TimeoutException` → locator or wait issue (test defect) |
| 4 | `AssertionError` → the application behaved differently (likely product defect) |
| 5 | `StaleElementReferenceException` → re-find the element after a DOM update (test defect) |
| 6 | Re-run the single test locally to confirm reproducibility |
| 7 | Raise a product bug **or** a test-maintenance ticket — never silently `@Test(enabled=false)` |

### 4.3 Suite health rules
A flaky test is a broken test. If a test fails intermittently twice in a sprint it is quarantined
into a `flaky` group with a ticket and a due date — it does not stay in the main gate.

---

## 5. Hands-on implementation (Windows)

### Step 1 — Install the toolchain

Install **JDK 21 LTS** (Eclipse Temurin or Oracle JDK 21), then set the environment variables:

```bat
:: Run in an elevated Command Prompt, then open a NEW terminal
setx /M JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-21"
setx /M PATH "%PATH%;%JAVA_HOME%\bin"
```

```bat
java -version
javac -version
echo %JAVA_HOME%
```

Install **Maven** (or use the IDE's bundled Maven) and verify:

```bat
mvn -version
```

Install **Eclipse IDE for Java Developers** or **IntelliJ IDEA Community**, plus Google Chrome.

> **No manual chromedriver download.** Selenium 4.6+ includes **Selenium Manager**, which
> resolves and downloads the matching driver automatically. The old
> `System.setProperty("webdriver.chrome.driver", ...)` step is obsolete.

### Step 2 — Create the Maven project

`pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.acme.qa</groupId>
  <artifactId>web-regression</artifactId>
  <version>1.0.0</version>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <selenium.version>4.27.0</selenium.version>
    <testng.version>7.10.2</testng.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.seleniumhq.selenium</groupId>
      <artifactId>selenium-java</artifactId>
      <version>${selenium.version}</version>
    </dependency>
    <dependency>
      <groupId>org.testng</groupId>
      <artifactId>testng</artifactId>
      <version>${testng.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
        <configuration>
          <suiteXmlFiles>
            <suiteXmlFile>testng.xml</suiteXmlFile>
          </suiteXmlFiles>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

### Step 3 — Base test class (driver lifecycle)

`src/test/java/com/acme/qa/base/BaseTest.java`:

```java
package com.acme.qa.base;

import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.WebDriverWait;
import org.testng.ITestResult;
import org.testng.annotations.*;

import java.io.File;
import java.nio.file.*;
import java.time.Duration;

public class BaseTest {

    protected WebDriver driver;
    protected WebDriverWait wait;
    protected static final String BASE_URL =
            System.getProperty("baseUrl", "https://www.saucedemo.com");

    @BeforeSuite(alwaysRun = true)
    public void suiteSetup() throws Exception {
        Files.createDirectories(Paths.get("target/screenshots"));
        System.out.println("=== Regression suite starting against " + BASE_URL + " ===");
    }

    @BeforeMethod(alwaysRun = true)
    public void setUp() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");
        // Selenium Manager resolves the driver binary automatically
        driver = new ChromeDriver(options);
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(5));
        driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(30));
        wait = new WebDriverWait(driver, Duration.ofSeconds(15));
        driver.get(BASE_URL);
    }

    @AfterMethod(alwaysRun = true)
    public void tearDown(ITestResult result) {
        if (result.getStatus() == ITestResult.FAILURE && driver != null) {
            captureEvidence(result.getName());
        }
        if (driver != null) {
            driver.quit();          // ALWAYS quit - close() leaks the driver process
        }
    }

    private void captureEvidence(String testName) {
        try {
            File src = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
            String stamp = String.valueOf(System.currentTimeMillis());
            Files.copy(src.toPath(),
                    Paths.get("target/screenshots", testName + "-" + stamp + ".png"),
                    StandardCopyOption.REPLACE_EXISTING);
            Files.writeString(Paths.get("target/screenshots", testName + "-" + stamp + ".html"),
                    driver.getPageSource());
            System.out.println("Evidence captured for failed test: " + testName);
        } catch (Exception e) {
            System.out.println("Could not capture evidence: " + e.getMessage());
        }
    }

    @AfterSuite(alwaysRun = true)
    public void suiteTeardown() {
        System.out.println("=== Regression suite finished ===");
    }
}
```

### Step 4 — Page Objects

`src/test/java/com/acme/qa/pages/LoginPage.java`:

```java
package com.acme.qa.pages;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import java.time.Duration;

public class LoginPage {

    private final WebDriver driver;
    private final WebDriverWait wait;

    // Locators live HERE and nowhere else
    private final By username  = By.id("user-name");
    private final By password  = By.id("password");
    private final By loginBtn  = By.id("login-button");
    private final By errorMsg  = By.cssSelector("h3[data-test='error']");

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(15));
    }

    public ProductsPage loginAs(String user, String pass) {
        wait.until(ExpectedConditions.visibilityOfElementLocated(username)).clear();
        driver.findElement(username).sendKeys(user);
        driver.findElement(password).clear();
        driver.findElement(password).sendKeys(pass);
        driver.findElement(loginBtn).click();
        return new ProductsPage(driver);
    }

    public String errorText() {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(errorMsg)).getText();
    }

    public boolean isLoaded() {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(loginBtn)).isDisplayed();
    }
}
```

`src/test/java/com/acme/qa/pages/ProductsPage.java`:

```java
package com.acme.qa.pages;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import java.time.Duration;
import java.util.List;

public class ProductsPage {

    private final WebDriver driver;
    private final WebDriverWait wait;

    private final By title       = By.className("title");
    private final By items       = By.cssSelector(".inventory_item");
    private final By sortSelect  = By.cssSelector("[data-test='product-sort-container']");
    private final By cartBadge   = By.cssSelector(".shopping_cart_badge");
    private final By cartLink    = By.cssSelector(".shopping_cart_link");
    private final By itemPrices  = By.cssSelector(".inventory_item_price");

    public ProductsPage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(15));
    }

    public boolean isLoaded() {
        return wait.until(ExpectedConditions.textToBePresentInElementLocated(title, "Products"));
    }

    public int productCount() {
        wait.until(ExpectedConditions.presenceOfElementLocated(items));
        return driver.findElements(items).size();
    }

    public void sortBy(String visibleText) {
        new Select(wait.until(ExpectedConditions.elementToBeClickable(sortSelect)))
                .selectByVisibleText(visibleText);
    }

    public List<Double> prices() {
        return driver.findElements(itemPrices).stream()
                .map(e -> Double.parseDouble(e.getText().replace("$", "")))
                .toList();
    }

    public void addToCart(String productName) {
        String id = "add-to-cart-" + productName.toLowerCase().replace(" ", "-");
        wait.until(ExpectedConditions.elementToBeClickable(By.id(id))).click();
    }

    public int cartCount() {
        try {
            return Integer.parseInt(driver.findElement(cartBadge).getText());
        } catch (NoSuchElementException e) {
            return 0;
        }
    }

    public void openCart() {
        wait.until(ExpectedConditions.elementToBeClickable(cartLink)).click();
    }
}
```

### Step 5 — The tests (TestNG lifecycle, groups, priorities, data providers)

`src/test/java/com/acme/qa/tests/LoginTests.java`:

```java
package com.acme.qa.tests;

import com.acme.qa.base.BaseTest;
import com.acme.qa.pages.*;
import org.testng.Assert;
import org.testng.annotations.*;

public class LoginTests extends BaseTest {

    @Test(priority = 1, groups = {"smoke", "regression"},
          description = "TC-01 Valid user can log in and reach the product catalogue")
    public void validLoginReachesCatalogue() {
        ProductsPage products = new LoginPage(driver)
                .loginAs("standard_user", "secret_sauce");
        Assert.assertTrue(products.isLoaded(), "Product catalogue did not load after login");
        Assert.assertTrue(driver.getCurrentUrl().contains("inventory"),
                "Expected to land on the inventory page");
    }

    @DataProvider(name = "invalidCredentials")
    public Object[][] invalidCredentials() {
        return new Object[][] {
            {"standard_user", "wrong_password", "Username and password do not match"},
            {"no_such_user",  "secret_sauce",   "Username and password do not match"},
            {"",              "secret_sauce",   "Username is required"},
            {"standard_user", "",               "Password is required"},
            {"locked_out_user", "secret_sauce", "Sorry, this user has been locked out"}
        };
    }

    @Test(priority = 2, groups = {"regression"}, dataProvider = "invalidCredentials",
          description = "TC-02 Invalid credential combinations are rejected with a clear message")
    public void invalidLoginShowsError(String user, String pass, String expectedFragment) {
        new LoginPage(driver).loginAs(user, pass);
        String actual = new LoginPage(driver).errorText();
        Assert.assertTrue(actual.contains(expectedFragment),
                "Expected error containing [" + expectedFragment + "] but got [" + actual + "]");
    }
}
```

`src/test/java/com/acme/qa/tests/CatalogueTests.java`:

```java
package com.acme.qa.tests;

import com.acme.qa.base.BaseTest;
import com.acme.qa.pages.*;
import org.testng.Assert;
import org.testng.annotations.*;

import java.util.List;

public class CatalogueTests extends BaseTest {

    private ProductsPage products;

    @BeforeMethod(alwaysRun = true, dependsOnMethods = "setUp")
    public void loginFirst() {
        products = new LoginPage(driver).loginAs("standard_user", "secret_sauce");
        Assert.assertTrue(products.isLoaded(), "Pre-condition failed: catalogue not loaded");
    }

    @Test(priority = 1, groups = {"smoke", "regression"},
          description = "TC-10 Catalogue displays the expected number of products")
    public void catalogueShowsAllProducts() {
        Assert.assertEquals(products.productCount(), 6, "Unexpected product count");
    }

    @Test(priority = 2, groups = {"regression"},
          description = "TC-11 Sorting by price low-to-high returns an ascending list")
    public void sortPriceLowToHigh() {
        products.sortBy("Price (low to high)");
        List<Double> prices = products.prices();
        List<Double> sorted = prices.stream().sorted().toList();
        Assert.assertEquals(prices, sorted, "Prices are not in ascending order");
    }

    @Test(priority = 3, groups = {"smoke", "regression"},
          description = "TC-12 Adding an item updates the cart badge")
    public void addItemUpdatesCartBadge() {
        Assert.assertEquals(products.cartCount(), 0, "Cart should start empty");
        products.addToCart("Sauce Labs Backpack");
        Assert.assertEquals(products.cartCount(), 1, "Cart badge did not update");
    }

    @Test(priority = 4, groups = {"regression"}, dependsOnMethods = "addItemUpdatesCartBadge",
          description = "TC-13 Cart page lists the added item")
    public void cartContainsAddedItem() {
        products.addToCart("Sauce Labs Backpack");
        products.openCart();
        Assert.assertTrue(driver.getPageSource().contains("Sauce Labs Backpack"),
                "Added item not present on the cart page");
    }
}
```

### Step 6 — `testng.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="WebRegressionSuite" verbose="1">

  <parameter name="baseUrl" value="https://www.saucedemo.com"/>

  <test name="SmokePack">
    <groups>
      <run><include name="smoke"/></run>
    </groups>
    <classes>
      <class name="com.acme.qa.tests.LoginTests"/>
      <class name="com.acme.qa.tests.CatalogueTests"/>
    </classes>
  </test>

  <test name="FullRegression">
    <groups>
      <run><include name="regression"/></run>
    </groups>
    <classes>
      <class name="com.acme.qa.tests.LoginTests"/>
      <class name="com.acme.qa.tests.CatalogueTests"/>
    </classes>
  </test>

</suite>
```

### Step 7 — Run it

```bat
mvn clean test

:: Override the environment under test
mvn clean test -DbaseUrl=https://staging.saucedemo.com

:: Run a single class
mvn test -Dtest=LoginTests

:: Reports
start target\surefire-reports\index.html
start target\surefire-reports\emailable-report.html
dir target\screenshots
```

### Step 8 — Prove the failure-evidence path works

Temporarily break a locator (e.g. change `By.id("user-name")` to `By.id("user-nameX")`), run the
suite, and confirm:

* the test fails with `NoSuchElementException`/`TimeoutException`,
* a PNG and an HTML page source land in `target/screenshots`,
* the TestNG report names the failing test and the exception.

Then restore the locator and re-run — all green.

---

## 6. Acceptance criteria

- [ ] `mvn clean test` runs the suite end to end on a clean machine
- [ ] No `Thread.sleep()` anywhere; all waits are explicit
- [ ] No `System.setProperty("webdriver...")` — Selenium Manager resolves the driver
- [ ] Locators exist only inside page classes
- [ ] At least 8 test cases including a `@DataProvider` with 5 data sets
- [ ] `smoke` and `regression` groups both runnable independently
- [ ] Screenshot + page source captured automatically on failure
- [ ] `driver.quit()` guaranteed — no orphaned `chrome.exe`/`chromedriver.exe` after a run
- [ ] Every test passes when run alone (independence check)

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `SessionNotCreatedException: version mismatch` | Old manual driver on PATH | Delete it; let Selenium Manager resolve the driver |
| `NoSuchElementException` | Wrong locator, or element not yet rendered | Verify in DevTools; use `WebDriverWait` |
| `ElementClickInterceptedException` | Overlay/banner on top | Wait for the overlay to disappear, or scroll into view |
| `StaleElementReferenceException` | DOM re-rendered after you found the element | Re-find the element after the action |
| `TimeoutException` on a slow page | Wait too short | Increase the explicit wait for that step only |
| Tests pass alone, fail in the suite | Shared state between tests | Fresh driver per test in `@BeforeMethod`; no static state |
| Browsers accumulate in Task Manager | `close()` instead of `quit()`, or exception before teardown | `quit()` in `@AfterMethod(alwaysRun = true)` |
| `JAVA_HOME is not set` | Env var missing or terminal not restarted | `setx /M JAVA_HOME ...`, open a new terminal |
| Maven can't download dependencies | Corporate proxy | Configure `~/.m2/settings.xml` proxy block |

---

## 8. Deliverables

1. Git repo: `pom.xml`, `testng.xml`, `src/test/java/...` (base, pages, tests).
2. `TEST-PLAN.md` — test case IDs, journeys covered, data sets, in/out of scope.
3. `QA-SOP-013.md` — design rules, failure triage procedure, flaky-test policy.
4. A passing run report plus one deliberately failing run with captured evidence.
5. Before/after table: 2 days manual vs suite execution time.

---

## 9. Interview talking points

* *"What is the Page Object Model and why use it?"* — Locators and page behaviour live in one
  class per page; when the UI changes you edit one file instead of every test.
* *"Implicit vs explicit wait?"* — Implicit is a global polling default; explicit waits for a
  specific condition on a specific element. Never mix them aggressively, and never use `sleep`.
* *"How do you handle flaky tests?"* — Find the root cause (timing, shared state, locator), fix
  it; quarantine with a ticket and a due date if it cannot be fixed immediately. Never disable it
  silently.
* *"How do you run the same test with many inputs?"* — `@DataProvider`; it reports each data set
  as its own result so you see exactly which input failed.
* *"`close()` vs `quit()`?"* — `close()` closes one window; `quit()` ends the session and kills
  the driver process. Always `quit()` in teardown.
* *"How do you make tests independent?"* — New driver per test method, no static shared state,
  each test sets up its own preconditions.

**CV bullet:** *Built a Selenium 4 + TestNG regression suite (Page Object Model, data-driven
tests, grouped smoke/regression execution, automatic screenshot and page-source capture on
failure) replacing a two-day manual regression cycle with a single Maven command.*
