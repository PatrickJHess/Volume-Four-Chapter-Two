



# Imported Functions and `FredReader`.

:::{dropdown} Click to see `FEDInvest`
```python
def FEDInvest(price_date):
  """
    Fetches historical security prices from the FedInvest portal.


    Args:
        price_date (datetime.date): The date for which to retrieve prices.
            Note: Current day is typically available after 1:00 PM ET on business days.




    Returns:
        tuple: (pandas.DataFrame, str) if successful. The DataFrame contains
               security details (CUSIP, Price, Yield), and the string is the
               official "Prices For" date stamp from the site.
        tuple: (str, None) if the request fails or no data is found for the date
                (attempt to fetch current day before 1:00 PM ET).


    Example:
        >>> from datetime import date
        >>> df, stamp = FEDInvest(date(2025, 3, 17))
  """


  def make_date(date_value):
    # datetime64 are conerted
    if not isinstance(date_value,(datetime,date)):
      try:
        date_value=pd.Timestamp(date_value).date()
      except Exception as e: # Catch anything else unexpected
        print(f"wrong type for settlement or maturity {e}")


      date_value=pd.Timestamp(settlement).date()
    # convert timestamps and datetimes to date
    else:
      try:
        date_value=date_value.date()
      except:
        pass
    return date_value


  price_date=make_date(price_date)
  # make share date of prices and settlement date are settlement dates
  price_date=adjust_bond_pay_dates(price_date)[0]
  if price_date > date.today():
    return "price_date is in the future", None, None


  settlement_date=price_date+relativedelta(days=1)
  settlement_date=adjust_bond_pay_dates(settlement_date)


  # URL address of Treasury Direct Select A Date
  url = "https://treasurydirect.gov/GA-FI/FedInvest/selectSecurityPriceDate"


  # Standard headers to look like a real browser
  headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36\
     (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Content-Type": "application/x-www-form-urlencoded"
  }
  #  variable names and type identified from inspecting url
  month=str(price_date.month)
  day=str(price_date.day)
  year=str(price_date.year)


  # payload passed in request post
  payload={'priceDate.month':month,
           'priceDate.day':day,
           'priceDate.year':year,
           "submit": "Show Prices"}


  # fires off form and returns prices for date
  try:
        response = requests.post(url, data=payload, headers=headers, timeout=10)
        response.raise_for_status()
  except requests.exceptions.RequestException as e:
        return f"Connection Error: {e}", None


  # reads the html
  # Pandas recommends to wrap the response in StingIO to make file like
  tables=pd.read_html(StringIO(response.text),match='CUSIP')


  # from inspection there is a single table
  df=tables[0]


  df['MATURITY DATE']=pd.to_datetime(df['MATURITY DATE'])


  # drop rows equal to or less than settlement date
  df_filtered=df[df['MATURITY DATE']>pd.to_datetime(settlement_date[0])]


  return df_filtered, price_date,settlement_date[0]

```
:::

:::{dropdown} Click to see `clean_FEDInvest`
```python
def clean_FEDInvest(df):


    import pandas as pd
    # Filters for Standard Securities
    keep_rows=df['SECURITY TYPE'].str.contains('bill|note|bond',case=False)
    security_df=df[keep_rows].copy()
 
    # Removes Clutter
    drop_columns=['CUSIP','CALL DATE']
    security_df.drop(columns=drop_columns,inplace=True)


    # Creates a Time-Series Index
    security_df.set_index('MATURITY DATE',inplace=True)
    security_df.index=pd.to_datetime(security_df.index)
    security_df.sort_index(inplace=True)


    # Standardizes Financial Terms
    change_column_names={'RATE':'Coupon',
                         'BUY':'Price Ask',
                         'SELL':'Price Bid'}
    security_df.rename(columns=change_column_names,inplace=True)


    # Formats Numeric Data
    numeric_cols = ['Coupon', 'Price Ask', 'Price Bid', 'YIELD']
    for col in numeric_cols:
        if col in security_df.columns:
            security_df[col] = security_df[col].astype(str).str.replace('%', '', regex=False).astype(float)


    return security_df

```
:::

:::{dropdown} Click to see `FredReader`

```py
class FredReader:
    def __init__(self, api_key=None, key_name="fred_key", cache_dir="fred"):
        # --- COLAB PERSISTENCE LOGIC ---
        if 'google.colab' in sys.modules:
            print("☁️ Colab environment detected. Mounting Google Drive...")
            from google.colab import drive
            try:
                drive.mount('/content/drive')
                self.cache_dir = '/content/drive/MyDrive/FRED'
            except Exception as e:
                print(f"✅ Drive mount failed, defaulting to local cache: {e}")
                self.cache_dir = cache_dir
        else:
            self.cache_dir = cache_dir

        self.api_key = None
        
        # This list prioritizes their custom name, then tries all standard variations
        fallback_names = [key_name, "fred_key", "FRED_API_KEY", "FRED_KEY"]

        # --- 1. EXPLICIT KEY PASSED ---
        if api_key:
            self.api_key = api_key

        # --- 2. CHECK COLAB SECRETS ---
        elif 'google.colab' in sys.modules:
            try:
                from google.colab import userdata
                for name_to_check in fallback_names:
                    try:
                        # Colab throws an error if the secret doesn't exist, so we catch it
                        potential_key = userdata.get(name_to_check)
                        if potential_key:
                            self.api_key = potential_key
                            print(f"✅ Key loaded seamlessly from Colab Secrets ('{name_to_check}')")
                            break # Stop searching, we found it!
                    except Exception:
                        continue # Secret not found, try the next name in the list
            except ImportError:
                pass 

        # --- 3. CHECK OS ENVIRONMENT (Local/Binder/Setup Wizard) ---
        if not self.api_key:
            for name_to_check in fallback_names:
                if os.environ.get(name_to_check):
                    self.api_key = os.environ.get(name_to_check)
                    print(f"✅ Key loaded from local environment ('{name_to_check}')")
                    break # Stop searching, we found it!

        # --- 4. FALLBACK PROMPT ---
        if not self.api_key:
            print(f"⚠️ Could not find a key named '{key_name}' (or standard fallbacks) in Colab Secrets or local environment.")
            print("If you have a FRED key, paste it now; otherwise just press Enter:")
            key_input = getpass.getpass(prompt="> ")
            if key_input.strip():
                self.api_key = key_input.strip()
                os.environ[key_name] = self.api_key # Save it for later cells using their preferred name!
                print("✅ Key loaded successfully from manual input!")
            else:
                print("⚠️ No key entered. Defaulting to pandas_datareader.")

        if not os.path.exists(self.cache_dir):
            os.makedirs(self.cache_dir)

    # --- REACTIVE NETWORK SHIELD ---
    def _make_api_request(self, url, max_retries=3, series_id=None):
        """Universal helper to handle API calls, 429 Too Many Requests, and graceful 400 errors."""
        import time
        import requests
        
        for attempt in range(max_retries):
            try:
                response = requests.get(url)
            except requests.exceptions.RequestException:
                print(f"🌩️ NETWORK ERROR: Failed to reach FRED servers.")
                return None
            
            # 1. Rate Limit Catch (HTTP 429)
            if response.status_code == 429:
                wait_time = 5 * (attempt + 1)
                print(f"🚦 Rate limit hit (HTTP 429)! Sleeping for {wait_time}s (Attempt {attempt + 1}/{max_retries})...")
                time.sleep(wait_time)
                continue
                
            # 2. Graceful Bad Request Catch (HTTP 400)
            if response.status_code == 400:
                try:
                    error_msg = response.json().get("error_message", "Unknown FRED API error")
                except ValueError:
                    error_msg = "Invalid request (No JSON message provided by FRED)."
                
                context_str = f" for '{series_id}'" if series_id else ""
                print(f"❌ API REJECTED{context_str}: {error_msg}")
                return None  
                
            # 3. Hard Crash Prevention for other HTTP errors (like 500 Server Error)
            try:
                response.raise_for_status()
            except requests.exceptions.HTTPError as e:
                print(f"🌩️ HTTP ERROR: {e}")
                return None

            # 4. Success! Safely unpack the JSON.
            try:
                return response.json()
            except ValueError:
                print(f"❌ ERROR: FRED returned a successful status, but the data is not valid JSON.")
                return None
            
        print(f"❌ Max retries ({max_retries}) exceeded. The API is strictly rate-limiting you.")
        return None

    # --- PUBLIC WRAPPER METHOD ---
    def get_series(self, series_ids, start_date=None, end_date=None, ttl_days=7):
        """Fetches one or more series and returns them in a single merged DataFrame."""
        if isinstance(series_ids, str):
            series_ids = [series_ids]
        elif not hasattr(series_ids, '__iter__'):
            raise TypeError("series_ids must be a string or an iterable of strings.")

        dataframes = []

        for series_id in series_ids:
            print(f"\n☁️--- Processing {series_id} ---")

            df = self._get_single_series(series_id, start_date=start_date, end_date=end_date, ttl_days=ttl_days)
            if df is not None and not df.empty:
                dataframes.append(df)
            else:
                print(f"⚠️ Skipping {series_id}: No data was returned.")

        if dataframes:
            if len(dataframes) == 1:
                return dataframes[0]
            print("\n🧩 Merging all series into a single DataFrame...")
            combined_df = pd.concat(dataframes, axis=1, join='outer')
            combined_df.sort_index(inplace=True)
            print("✅ Merge complete!")
            return combined_df
        else:
            print("❌ No data could be retrieved.")
            return None

    # --- CORE WORKHORSE METHOD ---
    def _get_single_series(self, series_id, start_date=None, end_date=None, ttl_days=7):
        import datetime as dt
        series_dir = os.path.join(self.cache_dir, series_id)

        filepath = os.path.abspath(os.path.join(series_dir, f"{series_id}_fred.csv"))
        metadata_file = os.path.abspath(os.path.join(series_dir, "cache_metadata.json"))

        metadata = {}
        metadata_is_stale = True
        metadata_updated_this_run = False

        # --- NESTED METADATA HELPER ---
        def update_metadata():
            nonlocal metadata, metadata_updated_this_run
            if not self.api_key: return True
            
            meta_url = f"https://api.stlouisfed.org/fred/series?series_id={series_id}&api_key={self.api_key}&file_type=json"
            try:
                meta_data = self._make_api_request(meta_url, series_id=series_id)
                
                if not meta_data:
                    return False

                if 'seriess' in meta_data and len(meta_data['seriess']) > 0:
                    series_info = meta_data['seriess'][0]
                    metadata['series_inception'] = series_info.get('observation_start')
                    metadata['series_last_observed'] = series_info.get('observation_end')
                    metadata['title'] = series_info.get('title')
                    metadata['frequency'] = series_info.get('frequency')
                    metadata['last_updated'] = dt.datetime.now(dt.timezone.utc).isoformat()
                    
                    os.makedirs(series_dir, exist_ok=True)

                    # Note: Because 'metadata' is modified in place, existing cache_start/cache_end
                    # are perfectly preserved here. We do not change them during a routine ping.
                    with open(metadata_file, 'w') as f:
                        json.dump(metadata, f, indent=4)
                    
                    metadata_updated_this_run = True
                    print("⏳ Metadata synced and timestamp updated.")
                    return True
            except Exception as e:
                print(f"⚠️ Could not update metadata: {e}")
                return False

        # --- 1. READ LOCAL METADATA ---
        if os.path.exists(metadata_file):
            try:
                with open(metadata_file, 'r') as f:
                    metadata = json.load(f)
            except (json.JSONDecodeError, ValueError) as e:
                print(f"⚠️ Warning: Corrupt or empty metadata cache found for {series_id}. Resetting file...")
                metadata = {}
                try:
                    os.remove(metadata_file) 
                except Exception:
                    pass
                
            if 'last_updated' in metadata:
                last_updated = dt.datetime.fromisoformat(metadata['last_updated'])
                if last_updated.tzinfo is None:
                    last_updated = last_updated.replace(tzinfo=dt.timezone.utc)
                
                days_old = (dt.datetime.now(dt.timezone.utc) - last_updated).days
                
                if days_old < ttl_days:
                    metadata_is_stale = False
                    print(f"🕒 Metadata is fresh ({days_old} days old).")
                else:
                    print(f"⏳ Metadata is {days_old} days old (TTL: {ttl_days}). It's stale.")

        # --- 2. SMART PING ---
        if not metadata:
            print(f"🆕 First run for {series_id}. Initializing metadata...")
            if not update_metadata():
              return None

        elif self.api_key and end_date and 'series_last_observed' in metadata and metadata_is_stale:
            if pd.to_datetime(end_date) > pd.to_datetime(metadata['series_last_observed']):
                print(f"🔎 Requested date exceeds known end date. Checking for updates...")
                update_metadata()

        # --- 3. CLAMP DATES ---
        if 'series_inception' in metadata and start_date:
            clamped_start = max(pd.to_datetime(start_date), pd.to_datetime(metadata['series_inception']))
            if pd.to_datetime(start_date) < clamped_start:
                start_date = clamped_start.strftime('%Y-%m-%d')
                print(f'⚠️ Adjusted start date to First Available Data: {start_date}')

        if 'series_last_observed' in metadata and end_date:
            clamped_end = min(pd.to_datetime(end_date), pd.to_datetime(metadata['series_last_observed']))
            if pd.to_datetime(end_date) > clamped_end:
                end_date = clamped_end.strftime('%Y-%m-%d')
                print(f'⚠️ FRED has no data past {end_date}. Adjusted request to match.')

        # --- 4. CHECK LOCAL CACHE ---
        cache_valid = False
        cache_start = None
        cache_end = None

        if os.path.exists(filepath):
            cache_valid = True

            if metadata_is_stale:
                cache_valid = False
                print(f"♻️ TTL expired. Forcing data and metadata refresh for {series_id}...")

            if cache_valid:
                if 'cache_start' in metadata and 'cache_end' in metadata:
                    cache_start = pd.to_datetime(metadata['cache_start'])
                    cache_end = pd.to_datetime(metadata['cache_end'])
                else:
                    try:
                        temp_df = pd.read_csv(filepath, index_col='DATE', parse_dates=True)
                        if not temp_df.empty:
                            cache_start = temp_df.index.min()
                            cache_end = temp_df.index.max()
                        else:
                            cache_valid = False
                    except Exception:
                        cache_valid = False

                if cache_valid and cache_start is not None:
                    if start_date and pd.to_datetime(start_date) < cache_start:
                        if len(pd.bdate_range(start_date, cache_start, inclusive='left')) > 0:
                            cache_valid = False

                    if end_date and pd.to_datetime(end_date) > cache_end:
                        freq_str = metadata.get('frequency', '')
                        if 'Annual' in freq_str:
                            cache_end_coverage = cache_end + pd.DateOffset(years=1) - pd.Timedelta(days=1)
                        elif 'Quarterly' in freq_str:
                            cache_end_coverage = cache_end + pd.DateOffset(months=3) - pd.Timedelta(days=1)
                        elif 'Monthly' in freq_str:
                            cache_end_coverage = cache_end + pd.DateOffset(months=1) - pd.Timedelta(days=1)
                        elif 'Weekly' in freq_str:
                            cache_end_coverage = cache_end + pd.Timedelta(days=6)
                        else:
                            cache_end_coverage = cache_end 
                        
                        if pd.to_datetime(end_date) > cache_end_coverage:
                            cache_valid = False

            if cache_valid:
                print(f"✅ Loaded {series_id} from local cache.")
                df = pd.read_csv(filepath, index_col='DATE', parse_dates=True)
                if start_date:
                    df = df[df.index >= pd.to_datetime(start_date)]
                if end_date:
                    df = df[df.index <= pd.to_datetime(end_date)]
                return df
            else:
                print(f"♻️ Cache missing or insufficient bounds. Killing cache for {series_id}...")
                try:
                    os.remove(filepath)
                except OSError:
                    pass
                
                # 🛠️ CRITICAL FIX: If we destroy the CSV, we MUST clear the start/end dates 
                # from the metadata immediately so we don't leave phantom dates behind.
                keys_removed = False
                if 'cache_start' in metadata:
                    del metadata['cache_start']
                    keys_removed = True
                if 'cache_end' in metadata:
                    del metadata['cache_end']
                    keys_removed = True
                
                if keys_removed:
                    with open(metadata_file, 'w') as f:
                        json.dump(metadata, f, indent=4)

        # --- 5. API FETCH ---
        print(f"☁️ Fetching fresh {series_id} observations...")
        if not metadata_updated_this_run and metadata_is_stale:
            if not update_metadata():
              return None

        try:
            if self.api_key is None:
                print("⚠️ No API key found. Defaulting to pandas_datareader...")
                df = web.DataReader(series_id, 'fred', start=start_date, end=end_date)
            else:
                url = f"https://api.stlouisfed.org/fred/series/observations?series_id={series_id}&api_key={self.api_key}&file_type=json"
                if start_date: url += f"&observation_start={start_date}"
                if end_date: url += f"&observation_end={end_date}"

                safe_url = url.replace(self.api_key, "HIDDEN_KEY")
                print(f"📡 Sending URL to FRED: {safe_url}")

                data = self._make_api_request(url)

                if not data or 'observations' not in data:
                  print(f"❌ API Error: Could not retrieve observation data for {series_id}.")
                  return None

                df = pd.DataFrame(data['observations'])
                df['DATE'] = pd.to_datetime(df['date'])
                df[series_id] = pd.to_numeric(df['value'], errors='coerce')
                df = df.set_index('DATE')
                df = df[[series_id]]

            df.dropna(inplace=True)
            
            os.makedirs(series_dir, exist_ok=True)            
 
            df.to_csv(filepath)
            
            # 🛠️ CRITICAL FIX: We ONLY change the start and end dates here, 
            # proving we have actually replaced the cache file successfully.
            if not df.empty:
                metadata['cache_start'] = df.index.min().strftime('%Y-%m-%d')
                metadata['cache_end'] = df.index.max().strftime('%Y-%m-%d')
                with open(metadata_file, 'w') as f:
                    json.dump(metadata, f, indent=4)
                    
            print(f"Success! Data saved to {filepath}.")
            return df

        except Exception as e:
            print(f"❌ An error occurred: {e}")
            return None```
:::


:::{dropdown} Click to see `secure_key_setup`

```py
def secure_key_setup(key_name="FRED_KEY"):
    """
    The master UI for setting up API keys. Handles Colab, Local Jupyter,
    and Ephemeral (Binder) environments automatically, with live validation.
    """
    try:
        from IPython.display import clear_output, display, HTML
    except ImportError:
        clear_output = lambda: None
        display = lambda x: print(x)
        HTML = lambda x: x

    # --- INTERNAL VALIDATION HELPER ---
    def _validate_key(name, value):
        import requests
        name_upper = name.upper()

        if "FRED" in name_upper:
            print(f"🔒 Authenticating {name} with FRED servers...")
            if len(value) != 32: return False
            try:
                url = f"https://api.stlouisfed.org/fred/series?series_id=GDP&api_key={value}&file_type=json"
                return requests.get(url).status_code == 200
            except Exception: return False

        elif "ALPHA" in name_upper:
            print(f"🔒 Authenticating {name} with AlphaVantage...")
            try:
                url = f"https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=IBM&apikey={value}"
                resp = requests.get(url).json()
                return "Error Message" not in resp
            except Exception: return False

        elif "MASSIVE" in name_upper:
            print(f"🔒 Authenticating {name} with Massive reference servers...")
            try:
                url = f"https://api.massive.com/v3/reference/tickers?limit=1&apiKey={value}"
                return requests.get(url).status_code == 200
            except Exception: return False

        else:
            # The graceful bypass for custom keys
            print(f"ℹ️ Note: '{name}' is not recognized as FRED, ALPHA, or MASSIVE.")
            print(f"   Skipping live network validation. Key loaded directly.")
            return True

    # 1. Detect Environment
    import sys, os, getpass
    in_colab = 'google.colab' in sys.modules
    in_binder = 'BINDER_URL' in os.environ or 'BINDER_PORT' in os.environ

    class StopExecution(Exception):
        def _render_traceback_(self):
            pass

    if in_colab:
        from google.colab import userdata
        from google.colab import output

        status_msg = "" # Used to display errors if they click the button too early

        while True:
            try:
                # STATE 1: Vault Check
                colab_key = userdata.get(key_name)
                if colab_key:
                    clear_output()
                    if _validate_key(key_name, colab_key):
                        os.environ[key_name] = colab_key
                        clear_output()
                        display(HTML(f"""
                        <div style="background-color: #d4edda; padding: 15px; border-radius: 8px; border-left: 6px solid #28a745; max-width: 600px; font-family: sans-serif;">
                            <h4 style="margin-top: 0; color: #155724; margin-bottom: 5px;">&#10004;&#65039; Key Authenticated & Active!</h4>
                            <p style="margin-top: 0; color: #155724; margin-bottom: 0;">We verified <b>{key_name}</b> from your Secrets and securely loaded it. You are ready to fetch data!</p>
                        </div>
                        """))
                        return
                    else:
                        status_msg = f"""
                        <div style="background-color: #f8d7da; padding: 15px; border-radius: 8px; border-left: 6px solid #dc3545; max-width: 600px; font-family: sans-serif; margin-bottom: 15px;">
                            <h4 style="margin-top: 0; color: #721c24; margin-bottom: 5px;">&#10060; Invalid Key Detected</h4>
                            <p style="margin-top: 0; color: #721c24; margin-bottom: 0;">We found <b>{key_name}</b>, but authentication failed. Please fix any typos in the sidebar.</p>
                        </div>
                        """
            except Exception as e:
                # STATE 2: Locked (NotebookAccessError)
                if "Access" in str(type(e).__name__):
                    status_msg = f"""
                    <div style="background-color: #fff3cd; padding: 15px; border-radius: 8px; border-left: 6px solid #ffc107; max-width: 600px; font-family: sans-serif; margin-bottom: 15px;">
                        <h4 style="margin-top: 0; color: #856404; margin-bottom: 5px;">&#128273; Activation Required</h4>
                        <p style="margin-top: 0; color: #856404; margin-bottom: 0;">We found <b>{key_name}</b>, but you forgot to toggle <b>"Notebook access"</b> to ON. Please toggle it and try again.</p>
                    </div>
                    """
                else:
                    pass # Doesn't exist yet, which is perfectly normal.

            # --- STATE 3: THE SEAMLESS SETUP WIZARD ---
            clear_output()
            if status_msg:
                display(HTML(status_msg))
                status_msg = "" # Clear it so it doesn't loop forever

            display(HTML(f"""
            <div style="font-family: sans-serif; max-width: 600px; border: 1px solid #ddd; border-radius: 8px; overflow: hidden; margin-bottom: 15px;">
                <div style="background-color: #f8f9fa; padding: 15px; border-bottom: 1px solid #ddd;">
                    <h3 style="margin: 0; color: #1a73e8;">&#128274; {key_name} Setup Required</h3>
                </div>
                <div style="padding: 20px;">
                    <p style="margin-top: 0; margin-bottom: 15px;">Follow these fast steps to securely store your API key:</p>
                    <ol style="margin-top: 0; padding-left: 20px; line-height: 1.8; color: #333;">
                        <li>Click the <b>Key Icon</b> (&#128273;) on the left sidebar.</li>
                        <li>Click <b>"Add new secret"</b>.</li>
                        <li>Paste your actual API Key into the <b>"Value"</b> box first (since it's on your clipboard!).</li>
                        <li>Copy this exact name: <button onclick="navigator.clipboard.writeText('{key_name}'); this.innerHTML='&#9989; Copied!'; setTimeout(() => this.innerHTML='&#128203; {key_name}', 2000);" style="margin-left: 8px; padding: 4px 8px; background: #e8f0fe; color: #1a73e8; border: 1px solid #1a73e8; border-radius: 4px; cursor: pointer; font-family: monospace; font-weight: bold;">&#128203; {key_name}</button><br>and paste it into the <b>"Name"</b> box.</li>
                        <li>Toggle <b>"Notebook access"</b> to <span style="color: #1a73e8; font-weight: bold;">ON (Blue)</span>.</li>
                    </ol>
                    
                    <div style="display: flex; gap: 10px; margin-top: 15px;">
                        <button id="check-btn-{key_name}" style="flex: 2; padding: 12px; background: #34a853; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 16px; font-weight: bold; transition: 0.2s;">
                            &#10004;&#65039; Authenticate Now
                        </button>
                        <button id="stop-btn-{key_name}" style="flex: 1; padding: 12px; background: #dc3545; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 16px; font-weight: bold; transition: 0.2s;">
                            &#10060; Stop
                        </button>
                    </div>
                </div>
            </div>

<script>
              // The Promise now listens to BOTH buttons and returns a command string
              window.promise_{key_name} = new Promise(function(resolve) {{
                
                document.getElementById('check-btn-{key_name}').onclick = function() {{
                  this.innerHTML = "&#8987; Authenticating...";
                  this.style.backgroundColor = "#2d9249";
                  this.style.pointerEvents = "none";
                  document.getElementById('stop-btn-{key_name}').style.pointerEvents = "none";
                  resolve("check"); // Signal Python to loop and check again
                }};
                
                document.getElementById('stop-btn-{key_name}').onclick = function() {{
                  this.innerHTML = "Stopping...";
                  this.style.backgroundColor = "#c82333";
                  this.style.pointerEvents = "none";
                  document.getElementById('check-btn-{key_name}').style.pointerEvents = "none";
                  resolve("stop"); // Signal Python to halt execution
                }};
                
              }});
            </script>
            """))
            
            # --- THE PAUSE TRAP ---
            try:
                # Python sleeps here waiting for the JS string response
                action = output.eval_js(f"window.promise_{key_name}")
                
                # If the user clicked Stop, break the script entirely
                if action == "stop":
                    clear_output()
                    print("\n\u26A0\uFE0F Setup cancelled by user.")
                    raise StopExecution
                    
            except KeyboardInterrupt:
                clear_output()
                print("\n\u26A0\uFE0F Setup cancelled by user (KeyboardInterrupt).")
                raise StopExecution
    else:
        # --- JUPYTER / BINDER LOGIC ---
        # 1. Define what we are looking for
        filename = f".{key_name.lower()}"
        
        # 2. Start at the current working directory
        current_dir = os.path.abspath(os.getcwd())
        found_file_path = None

        # 3. Crawl UP the folder structure
        while True:
            potential_path = os.path.join(current_dir, filename)
            
            if os.path.exists(potential_path):
                found_file_path = potential_path
                break  # We found it! Stop searching.
                
            # Move up one level
            parent_dir = os.path.dirname(current_dir)
            
            # Check if we hit the root of the hard drive (e.g., C:\ or /)
            if current_dir == parent_dir:
                break  # Stop searching, it doesn't exist anywhere above us
                
            current_dir = parent_dir

        # 4. Set the final target_file path
        if found_file_path:
            target_file = found_file_path # Use the file we found up the tree
        else:
            target_file = os.path.join(os.getcwd(), filename) # Default to saving in cur

        # UI: Warning if existing file found (Local only)
        if not in_binder and os.path.exists(target_file):
            display(HTML(f"""<div style="background-color: #fff3cd; padding: 15px; border-radius: 8px; border-left: 6px solid #ffc107; font-family: sans-serif; max-width: 600px; margin-bottom: 10px;"><h4 style="margin-top: 0; color: #856404; margin-bottom: 5px;">&#9888;&#65039; WARNING: Key Already Exists</h4><p style="margin-top: 5px; color: #856404; margin-bottom: 0;">A saved key was already found in your vault.</p></div>"""))
            try:
                confirm = input("Do you want to OVERWRITE it? [y/N]: ")
                if confirm.strip().lower() not in ['yes', 'y']:
                    with open(target_file, "r") as f:
                        os.environ[key_name] = f.read().strip()
                    clear_output()
                    display(HTML("""<div style="margin-top: 10px; color: #155724; font-weight: bold; background: #d4edda; padding: 15px; border-radius: 8px; border-left: 6px solid #28a745; max-width: 600px;">&#10005; Setup cancelled. Your existing key was retained <b>and loaded into the environment!</b></div>"""))
                    return
            except KeyboardInterrupt:
                with open(target_file, "r") as f:
                    os.environ[key_name] = f.read().strip()
                clear_output()
                display(HTML("""<div style="margin-top: 10px; color: #155724; font-weight: bold; background: #d4edda; padding: 15px; border-radius: 8px; border-left: 6px solid #28a745; max-width: 600px;">&#10005; Interrupted. Your existing key was retained <b>and loaded into the environment!</b></div>"""))
                return
            clear_output()

        # UI: The Request Box
        if in_binder:
             display(HTML(f"""<div style="background-color: #e2e3e5; padding: 15px; border-radius: 8px; border-left: 6px solid #6c757d; font-family: sans-serif; max-width: 600px; margin-bottom: 10px;"><h3 style="margin-top: 0; color: #383d41; margin-bottom: 5px;">&#9201;&#65039; Ephemeral Session Setup</h3><p style="margin-top: 0; margin-bottom: 0;">You are running in a temporary session. Please paste your <b>{key_name}</b> below.<br><br><b>Note:</b> This key will only persist as long as this browser session remains active.</p></div>"""))
        else:
             display(HTML(f"""<div style="background-color: #f8f9fa; padding: 15px; border-radius: 8px; border-left: 6px solid #4285f4; font-family: sans-serif; max-width: 600px; margin-bottom: 10px;"><h3 style="margin-top: 0; color: #1a73e8; margin-bottom: 5px;">&#128274; Secure Key Setup</h3><p style="margin-top: 0; margin-bottom: 0;">Please paste your <b>{key_name}</b> below. It will be safely vaulted as a hidden file.</p></div>"""))

        # The Input Loop
        print(f"Enter your {key_name} (or type 'quit' to cancel):")
        while True:
            try:
                key_input = getpass.getpass(prompt="> ").strip()
                if key_input.lower() in ['q', 'quit', 'cancel', 'exit']:
                    clear_output()
                    print("\n\u26A0\uFE0F Setup cancelled by user. No key was saved.")
                    return

                if key_input:
                    # VALIDATION INTERCEPT
                    if _validate_key(key_name, key_input):
                        clear_output()
                        break
                    else:
                        clear_output()
                        print(f"\u274C Invalid {key_name} detected! Authentication failed.")
                        print(f"Please check for typos and try again (or type 'quit' to cancel):")
                else:
                    clear_output()
                    print("\u26A0\uFE0F Input cannot be empty. Please paste your key (or type 'quit' to cancel):")
            except KeyboardInterrupt:
                clear_output()
                print("\n\u26A0\uFE0F Cell execution interrupted. Setup cancelled.")
                return

        # INJECT INTO ENVIRONMENT IMMEDIATELY
        os.environ[key_name] = key_input

        # Save Logic (Skip saving to disk if Binder)
        if in_binder:
            display(HTML(f"""<div style="margin-top: 10px; color: #137333; font-weight: bold; background: #e6f4ea; padding: 15px; border-radius: 8px; border-left: 6px solid #34a853; max-width: 600px;">&#127881; <b>Success! Key authenticated and loaded to environment.</b><br><span style="color: #0d652d; font-size: 0.9em;">(Remember: It will be cleared when this session ends.)</span></div>"""))
        else:
            try:
                with open(target_file, "w") as f: f.write(key_input)
                try: os.chmod(target_file, 0o600)
                except: pass
                display(HTML(f"""<div style="margin-top: 10px; color: #137333; font-weight: bold; background: #e6f4ea; padding: 15px; border-radius: 8px; border-left: 6px solid #34a853; max-width: 600px;">&#127881; <b>Success! Your key has been verified and safely vaulted in <code>{target_file}</code></b><br><span style="color: #0d652d; font-size: 0.9em;">&#128161; <b>Pro Tip:</b> Future notebooks will automatically load it!</span></div>"""))
            except Exception as e:
                clear_output()
                print(f"\n\u274C Error saving key: {e}")


def load_key_to_env(key_name="FRED_KEY"):
    """
    Silently loads the API key. If it fails, acts as a router to the Setup UI.
    """
    if os.environ.get(key_name): return

    in_colab = 'google.colab' in sys.modules
    if in_colab:
        try:
            from google.colab import userdata
            colab_key = userdata.get(key_name)
            if colab_key:
                os.environ[key_name] = colab_key
                return
        except: pass 

    dot_file = f".{key_name.lower()}"
    if os.path.exists(dot_file):
        with open(dot_file, "r") as f:
            saved_key = f.read().strip()
            if saved_key:
                os.environ[key_name] = saved_key
                return

    # IF ALL FAILS -> Route to Setup UI
    secure_key_setup(key_name)
    
    # The Colab Trap
    if in_colab:
        raise RuntimeError(f"[\u26A0\uFE0F] Setup required! Please complete the wizard above, then re-run this cell.")
```

:::

:::{dropdown} Click to see `accrued_interest`

```py
def accrued_interest(maturity, coupon, day_type='Actual/Actual', settlement=None, freq=2):
    """
      Returns the accrued interest for a bond.  Returns the accrued interest for a bond.


       Returns the accrued interest for a bond.  Returns the accrued interest for a bond.


    Args:
        maturity (datetime.date,timestamp, or datetime64): The maturity date of the bond.
        coupon (float): The annual coupon rate (e.g., 0.05 for 5%).
        day_types:
            Actual/Actual, Actual/365, Actual/360. and 30/360.
        settlement ((datetime.date,timestamp, or datetime64), optional):
         The settlement date. Defaults to today.
        freq (int, optional):
        Coupon frequency per year: 1 (annual). 2 (semi-annual)
                                    4 (quarterly), 12 (monthly)
                                    Defaults to 2.
    """
    from datetime import datetime,date
    import calendar
    from dateutil.relativedelta import relativedelta
    #Validate Data
    def validate_date(datetime_object):
      # check for datetime or date
      if not isinstance(datetime_object, (datetime, date)):
          raise TypeError("Input must be a datetime or date object.")
      # convert datetime to date
      if isinstance(datetime_object, datetime):
        datetime_object = datetime_object.date()
      return datetime_object
    maturity = validate_date(maturity)


    if settlement is None:
        settlement = date.today()
    else:
        settlement = validate_date(settlement)


    if freq not in [1, 2, 4, 12]:
        print(f"⚠️ Warning: Freq {freq} invalid. Assumed Semi-Annual (2).")
        freq = 2


    try:
        coupon = float(coupon)
        if coupon < 0: raise ValueError
    except:
        raise ValueError("Coupon must be a positive number.")


    # Define strategic_date
    # Get all the bond's potential payment dates in the next year
    mat_is_last=maturity.day==calendar.monthrange(maturity.year,maturity.month)[1]
    if mat_is_last:
      lastDay=calendar.monthrange(settlement.year+1,maturity.month)[1]
      strategic_date=date(settlement.year+1,maturity.month,lastDay)
    else:
     strategic_date=date(settlement.year+1,maturity.month,maturity.day)


    #The strategic date: minimum of actual and next year's maturity date
    strategic_date=min(maturity,strategic_date)
    pay_dates = scheduled_pay_dates(strategic_date, settlement=settlement, freq=freq)


    # Should sorted but check
    pay_dates.sort()


    # The first date after the settlement date is the next coupon date
    next_coupon = None
    for d in pay_dates:
        if d >= settlement:
            next_coupon = d
            break


    # Bond has matured or annual coupon is zero
    if next_coupon is None or coupon==0:
        return 0.0


    #Calculate Previous Coupon Date
    num_months = int(12 // freq)
    prev_coupon = next_coupon - relativedelta(months=num_months)


    # Check for Month End adjustment on the calculated previous date
    is_next_month_end = next_coupon.day == calendar.monthrange(maturity.year, maturity.month)[1]


    if is_next_month_end:
        last_day_of_prev_month = calendar.monthrange(prev_coupon.year, prev_coupon.month)[1]
        prev_coupon = date(prev_coupon.year, prev_coupon.month, last_day_of_prev_month)


    # The day a coupon is paid is also the first day of the new cycle.


    accrued_value = 0.0


    if day_type == 'Actual/Actual':
        days_since_last = (settlement - prev_coupon).days
        days_between = (next_coupon - prev_coupon).days
        #  (DaysHeld / DaysInPeriod)
        accural_ratio= days_since_last/days_between
        # Actual/Actual uses the coupon paid on the date
        accrued_value = (coupon / freq) * accural_ratio




    elif day_type == '30/360':
        accural_ratio =_30_360_(settlement,prev_coupon)
        # Formula: Coupon * accrural ratio
        accrued_value = coupon * accural_ratio


    elif day_type == 'Actual/360':
        days_since_last = (settlement - prev_coupon).days
        accural_ratio= days_since_last/360
      # Formula: Coupon * accrural ratio        
        accrued_value = coupon * accural_ratio


    elif day_type == 'Actual/365':
        accural_ratio = convert_isda(settlement,prev_coupon)
      # Formula: Coupon * accrural ratio        
        accrued_value = coupon * accural_ratio


    else:
        # Fallback
        print(f"⚠️ Warning: Unknown day_type {day_type}. Using Actual/Actual.")
        days_since_last = (settlement - prev_coupon).days
        days_between = (next_coupon - prev_coupon).days
        #  (DaysHeld / DaysInPeriod)
        accural_ratio= days_since_last/days_between
        # Actual/Actual uses the coupon paid on the date
        accrued_value = (coupon / freq) * (days_since_last / days_between)


    return accrued_value

```
:::

:::{dropdown} Click to see `create_payoff_df`

```py
def create_payoff_df(df, settlement,OLS=False):
    adjusted_maturities = adjust_bond_pay_dates(list(df.index))
    all_maturities = set(adjusted_maturities)


    df_payoff_columns = sorted(all_maturities)
    df_payoff_index=[i for i in range(len(df.index))]


    df_payoff = pd.DataFrame(
        np.zeros((len(df), len(df_payoff_columns))),
        columns=df_payoff_columns,
        index=df_payoff_index
    )
    total_rows = len(df)
    # Define a clean, pleasing HTML template for our status box
    def status_box(current, total):
        return f"""
        <div style="font-family: Arial, sans-serif; padding: 10px 15px; background-color: #f8f9fa; 
                    border-left: 4px solid #007bff; border-radius: 4px; width: fit-content; color: #333;">
            <b style="color: #007bff;">⚙️ Processing Bonds:</b> {current} of {total} added to DataFrame
        </div>
        """
    
    # initial display showing 0 bonds added
    progress_ui = display(HTML(status_box(0, total_rows)), display_id=True)


    for index,(maturity, coupon) in enumerate(zip(df.index, df['Coupon'])):


        # bond_pay_data returns payment dates and amounts
        row_pay_data = bond_pay_data(maturity, coupon, settlement=settlement)


        # Find any dates that aren't already columns
        new_dates = set(row_pay_data[0].flatten()) - all_maturities


        if new_dates:
          if OLS:
            df_clean = df_payoff.loc[(df_payoff != 0).any(axis=1),
                                         (df_payoff != 0).any(axis=0)]
            print("\u2705 DataFrame Complete (Exited Early)!")
            return df_clean
          else:
            # "\u2705 FIX: Add new dates to our master set and reindex
            all_maturities.update(new_dates)
            df_payoff = df_payoff.reindex(columns=sorted(all_maturities), fill_value=0.0)
 
        #    fill up the columns
        df_payoff.loc[index, row_pay_data[0]] = row_pay_data[1]
         # update the progress bar
        progress_ui.update(HTML(status_box(index + 1, total_rows)))
    # re-sort the columns so dates are chronological
    df_payoff = df_payoff.reindex(sorted(df_payoff.columns), axis=1)
    progress_ui.update(HTML("""
        <div style="font-family: Arial, sans-serif; padding: 10px 15px; background-color: #e6f4ea; 
                    border-left: 4px solid #34a853; border-radius: 4px; width: fit-content; color: #137333;">
            <b>"\u2705 DataFrame Complete!</b> All bonds added successfully.
            </div>
            """))
    return df_payoff

```
:::

:::{dropdown} Click to see `ns_spot_rates`

```py
def ns_spot_rates(interim_estimates,mat_years,sofr_rate=None):
  """
    Calculates spot rates using the Nelson-Siegel yield curve model.
    
    Handles both the unrestricted 4-parameter model and a restricted 
    3-parameter model where the short end is pinned to a proxy rate (like SOFR).


    Args:
        interim_estimates (array-like): Current parameter estimates. 
            Expects 3 values (Beta 0, Beta 2, Tau) if sofr_rate is provided.
            Expects 4 values (Beta 0, Beta 1, Beta 2, Tau) if unrestricted.
        mat_years (array-like): Time to maturity for each cash flow, in years.
        sofr_rate (float, optional): The short-term rate to pin the curve to. 
            Defaults to None (triggers unrestricted 4-parameter model).


    Returns:
        tuple:
            - np.ndarray: The calculated spot rates for the given maturities.
            - np.ndarray: The adjusted time-to-maturity array (zeros replaced with 1e-8).
  """
  # t saves typing
  t=mat_years


  # Avoid division by zero for t=0
  t = np.where(t == 0, 1e-8, t)
   
  if sofr_rate is not None:  # SOFR ties download the short rate


    # current values of estimates
    b_0,b_2,tau=interim_estimates
 
    # Restricted Nelson-Siegel model Restricted
    spot_rates = (
            b_0 
            + (sofr_rate - b_0) * (1 - np.exp(-t/tau)) / (t/tau) 
            + b_2 * ((1 - np.exp(-t/tau)) / (t/tau) - np.exp(-t/tau)))
  else:    # SOFR ignored


    # Unrestricted Nelson-Siegel model Unrestricted


    # current values of estimates
    b_0,b_1,b_2,tau=interim_estimates


    spot_rates = (
            b_0 
            + b_1 * (1 - np.exp(-t/tau)) / (t/tau) 
            + b_2 * ((1 - np.exp(-t/tau)) / (t/tau) - np.exp(-t/tau))
        )
  # pass these rates to the objective function for step one
  return spot_rates,t

```
:::

:::{dropdown} Click to see `estimate_ns_parameters`

```py
def estimate_ns_parameters(df_payoff_matrix,P_actual,mat_years,guesses,sofr_rate=None):
  """
    Estimates Nelson-Siegel parameters by minimizing the Sum of Squared Residuals (SSR) 
    between actual bond prices and model-predicted bond prices.


    Args:
        df_payoff_matrix (pd.DataFrame or np.ndarray): Matrix of bond cash flows 
            where rows are bonds and columns represent time periods.
        P_actual (pd.Series or np.ndarray): Observed market prices of the bonds.
        mat_years (array-like): Time to maturity for each cash flow period, in years.
        guesses (list or array-like): Initial guesses for the optimization solver.
            Must have length 3 if sofr_rate is provided, or length 4 if None.
        sofr_rate (float, optional): A proxy short-term rate used to restrict the 
            short end of the curve. Defaults to None.


    Returns:
        scipy.optimize.OptimizeResult: The optimization result object containing 
            the estimated parameters (in the `.x` attribute), success status, 
            and minimum SSR achieved.


    Raises:
        ValueError: If the length of the `guesses` array does not match the 
            requirements of the chosen model type (restricted vs. unrestricted).
  """
  # objective function is sum squared residuals (SSR)
  def predict_prices(interim_estimates,df_payoff_matrix,P_actual,mat_years):


    # for step one get rates
    spot_rates,mat_years=ns_spot_rates(interim_estimates,mat_years,sofr_rate=sofr_rate)


    # calculate zero prices
    zero_prices=np.exp(-spot_rates*mat_years)


    # calculate predicted prices (@ here is the same as np.matmul)
    P_predicted=df_payoff_matrix@zero_prices


    # step 2 calculate the distance with sum of squared residuals
    ssr= np.sum((P_actual.values-P_predicted)**2)
    return ssr


  # use the scipy minimize function to estimate parameters


# Validate guess lengths based on model type
    if sofr_rate is not None and len(guesses) != 3:
        raise ValueError(f"Restricted model (SOFR={sofr_rate}) requires exactly 3 guesses. You provided {len(guesses)}.")
    
    if sofr_rate is None and len(guesses) != 4:
        raise ValueError(f"Unrestricted model requires exactly 4 guesses. You provided {len(guesses)}.")


  # Nelder-Mead doesn't take derivatives and tolerant of data
  method='Nelder-Mead'


  # minimize results
  ns_results=minimize(fun=predict_prices,
                  x0=guesses,
                  args=(df_payoff_matrix,P_actual,mat_years),
                  method=method)


  # get estimated coefficients of Nelson-Siegel




  # get status of minimization
  completion_status=ns_results.message


  if sofr_rate is not None:
    b_0,b_2,tau=ns_results.x
    display(f'Completion status: {completion_status}')
    display(f'Long Rate (Beta Zero) {b_0:.4f}..Slope (sofr_rate -Beta Zero) {sofr_rate-b_0: .4f}...\
    Shape (Beta Two) {b_2: .4f}...Scaling (Tau) {tau: .4f}') 


  else:
    b_0,b_1,b_2,tau=ns_results.x
    display(f'Completion status: {completion_status}')
    display(f'Long Rate (Beta Zero) {b_0:.4f}..Slope (Beta One) {b_1: .4f}...\
    Shape (Beta Two) {b_2: .4f}...Scaling (Tau) {tau: .4f}') 
  return ns_results

```
:::

:::{dropdown} Click to see `calc_par_yield`

```py
def calc_par_yield(months_to_maturity, interim_estimates, sofr_rate=None, freq=6, forward_start_months=0):
    """
    Calculates spot or forward par yields.
    
    Args:
        months_to_maturity (int/float): Tenor in months. Must be an int if >= freq.
        interim_estimates (list): Nelson-Siegel parameters.
        sofr_rate (float, optional): Short rate for restricted model.
        freq (int, optional): Payment frequency in months. Defaults to 6.
        forward_start_months (int/float, optional): Forward start period. Defaults to 0 (spot).
    """
    # 1. Guard against negative maturities
    if months_to_maturity < 0:
        raise ValueError(f"Maturity cannot be negative. Received: {months_to_maturity}")
        
    # 2. Type check: Enforce integers ONLY for swap-market maturities (>= freq)
    if months_to_maturity >= freq and not isinstance(months_to_maturity, int):
        raise TypeError(
            f"months_to_maturity must be an integer when >= freq ({freq} months). "
            f"Received: {type(months_to_maturity).__name__} ({months_to_maturity})"
        )
    
    # 3. Safeguard against exactly 0 maturity (estimates the immediate curve intercept)
    months_to_maturity = max(months_to_maturity, 0.0001)
    
    # 4. Generate schedule of relative months and accrual periods (gamma)
    if months_to_maturity < freq:
        # Short end: allows fractional months and single payments
        payment_months = [months_to_maturity]
        accrual_taus = [months_to_maturity / 12.0]
    else:
        # Short end: integer scheduling and sloppy stub handling
        full_periods = int(months_to_maturity // freq)
        payment_months = list(range(freq, (full_periods * freq) + 1, freq))
        accrual_taus = [freq / 12.0] * full_periods
        
        stub_months = months_to_maturity % freq
        if stub_months > 0:
            payment_months=[stub_months]+payment_months
            accrual_taus=[stub_months / 12.0]+accrual_taus            

    df = pd.DataFrame({
        'relative_months': payment_months,
        'tau': accrual_taus
    })
    
    # 5. Calculate Absolute Time (T) from TODAY (t=0)
    df['T_absolute'] = (df['relative_months'] + forward_start_months) / 12.0
    
    # 6. Get Spot Rates via your Nelson-Siegel function
    # if sofr_rate present check coeff
    if sofr_rate is not None and len(interim_estimates)>3:
      interim_estimates=list(interim_estimates)
      interim_estimates.pop(1)
    spot_rates, _ = ns_spot_rates(
        interim_estimates=interim_estimates, 
        mat_years=df['T_absolute'].values, 
        sofr_rate=sofr_rate
    )
    df['spot_rate'] = spot_rates
    
    # 7. Calculate Discount Factors back to today (t=0)
    df['DF'] = np.exp(-df['spot_rate'] * df['T_absolute'])
    
    # 8. Handle Forward Start Adjustment
    if forward_start_months > 0:
        T_start = forward_start_months / 12.0
        start_spot, _ = ns_spot_rates(interim_estimates, np.array([T_start]), sofr_rate)
        DF_start = np.exp(-start_spot[0] * T_start)
    else:
        DF_start = 1.0 
        
    # 9. Calculate Par Yield
    DF_N = df['DF'].iloc[-1]
    PV01 = (df['tau'] * df['DF']).sum()
    
    par_yield = (DF_start - DF_N) / PV01
    
    return par_yield
```
:::

:::{dropdown} Click to see `one_y_axis`

```py
def one_y_axis(x_data, y_data_list, title, series_labels, xlabel, ylabel,
                       markers, figure_size, y_limits,save_config={}, fill_config={},
                       colors=None):
    '''
    Plots data on a single y-axis.


    Args:
        x_data (array-like): Data for the x-axis.
        y_data_list (list of array-like): A list of datasets for the y-axis.
        title (str): The title of the graph.
        series_labels (list of str): Identifiers for each data series in the legend.
        xlabel (str): The label for the x-axis.
        ylabel (str): The label for the y-axis.
        markers (list of str): The markers to use for each series.
        figure_size (tuple): The width and height of the figure in inches.
        y_limits (tuple): The minimum and maximum values for the y-axis.
        save_config (dict, optional): Configuration for saving the file, passed
            to save_results(). Keys: 'volume', 'chapter', 'file_name'. Defaults to {}.
        fill_config (dict, optional): Configuration for filling areas.
            Keys: 'Between' (list of 1 or 2 indices from y_data_list),
                  'Start' (int, start index), 'End' (int, end index),
                  'Colors' (str), 'Labels' (str), 'Alpha' (float).
            Defaults to {}.
    Raises:
        ValueError: If input lists for series, markers, or colors do not match the number of y-datasets.
    '''
    import numpy as np
    from matplotlib import pyplot as plt
    num_series = len(y_data_list)
    # --- Input Validation ---
    if not all(len(lst) == num_series for lst in [series_labels, markers]):
        raise ValueError("The 'series_labels' and 'markers' lists must have the same length as 'y_data_list'.")


    if colors and len(colors) != num_series:
        raise ValueError("The 'colors' list must have the same length as 'y_data_list'.")


    # --- Plotting Setup ---
    fig = plt.figure(figsize=figure_size)
    fig.suptitle(title)
    plt.style.use('ggplot')


    if colors is None:
        # Generate a default color cycle if none are provided
        colors = plt.cm.viridis_r(np.linspace(0, 1, num_series))


# --- Plot Data Series ---
    for i in range(num_series):
        plt.plot(x_data, y_data_list[i], label=series_labels[i], marker=markers[i], color=colors[i])


    # --- Handle Fill Area ---
    if fill_config.get('Between'):
        if len(fill_config['Between']) > 2:
            raise ValueError("The 'Between' key in fill_config can contain a maximum of two indices.")




        # Get values from fill_config dict, providing safe defaults
        start = fill_config.get('Start', 0)
        end = fill_config.get('End', len(x_data))
        alpha = fill_config.get('Alpha', 0.3)
        color = fill_config.get('Colors', 'gray')
        label = fill_config.get('Labels', None) # 'None' won't create a legend item


        if len(fill_config['Between']) == 2:
            y1_index, y2_index = fill_config['Between']
            plt.fill_between(x_data[start:end],
                             y_data_list[y1_index][start:end],
                             y_data_list[y2_index][start:end],
                             color=color, alpha=alpha, label=label)
        else:
            y_index = fill_config['Between'][0]
            # Fills between the series and y=0
            plt.fill_between(x_data[start:end],
                             y_data_list[y_index][start:end],
                             color=color, alpha=alpha, label=label)


    # --- Final Touches ---
    plt.ylim(y_limits)
    plt.xlabel(xlabel)
    plt.ylabel(ylabel)
    plt.legend()
    plt.tight_layout()


    # --- Save Figure ---
    # Calls the save_results function (assumed to be defined)


    path = save_results(save_config=save_config)
    if path:
     plt.savefig(path, dpi=300, bbox_inches='tight')


    plt.show()

```
:::
