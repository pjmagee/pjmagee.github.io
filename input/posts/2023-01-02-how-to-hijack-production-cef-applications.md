Title: How to hi-jack CEF applications
Lead: Automate CEF applications that didn't disable debugging
Published: 2023-01-02
Tags:
  - 1Password
  - SWTOR
  - MMORPG
  - CEF
  - Chrome Debugger
---

It's important to ensure that you disable debugging functionality before releasing to production. Password protecting your CEF application exe can be bypassed by users who know which arguments they should pass to the exe, which can enable users to bypass the security you put in place.

Take the example below where the self contained isolated non Steam version of the Launcher for SWTOR does not protect against remote debugging, enabling the ability to extract full source code of the Launcher, reverse engineer the native interop to the C++ dll and also being able to automate the usage of the desktop application with a WebDriver.

```C#

async Task Main()
{	
	// use 1Password CLI to extract item you need for logging in

    var opCLI = @"C:\Users\patri\OneDrive\tools\op.exe";
    var args = "item get SWTOR --format json";
    var driverExeDir = @"C:\Users\patri\Downloads\chromedriver_win32";
    var cefClient = @"G:\Star Wars - The Old Republic\launcher.exe";

	using (var op = Process.Start(new ProcessStartInfo { FileName = opCLI, RedirectStandardOutput = true, Arguments = args }))
	{
		using(var document = JsonDocument.Parse(await op.StandardOutput.ReadToEndAsync()))
		{
			var items = document.RootElement
                .GetProperty("fields")
                .EnumerateArray()
                .Where(x => x.GetProperty("id").ValueEquals("username") || 
                            x.GetProperty("id").ValueEquals("password") || 
                            x.GetProperty("type").ValueEquals("OTP"))
                .ToArray();

			var details = new
			{
				username = items[0].GetProperty("value").GetString(),
				password = items[1].GetProperty("value").GetString(),
				otp = items[2].GetProperty("totp").GetString(),
			};

			// Chrome/75.0.3770.100
			// Download the Driver for the CEF client exe and configure additional arguments			
			var options = new ChromeOptions() { LeaveBrowserRunning = true, BinaryLocation = cefClient };
			options.AddArgument("--debug");
			options.AddArgument("--remote-debugging-port=9222");
			
			using (IWebDriver driver = new ChromeDriver(driverExeDir, options))
			{
				// Use Chrome/Edge and inspect to find the right elements you need to automate and then apply this to your code				
				driver.FindElement(By.XPath("//*[@id=\"usernameInput\"]")).Clear();				
				driver.FindElement(By.XPath("//*[@id=\"usernameInput\"]")).SendKeys(details.username);
				
				driver.FindElement(By.XPath("//*[@id=\"passwordInput\"]")).SendKeys(details.password);
				driver.FindElement(By.XPath("//*[@name=\"securitykey\"]")).SendKeys(details.otp);
				
				driver.FindElement(By.Id("nextButton")).Click();	
				
				await Task.Delay(1000);				
				driver.FindElement(By.Id("closeAlertsBtn")).Click();				
				await Task.Delay(2000);				
				driver.FindElement(By.ClassName("hitbox")).Click();
			}
		}
	}
}


```
