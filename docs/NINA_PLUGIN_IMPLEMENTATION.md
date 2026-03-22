# N.I.N.A Plugin - Implementation Details & Code Structure

## Project Structure & Build Configuration

### Directory Layout

```
SeeStar_NINA_Plugin/
│
├── src/
│   ├── SeeStar_NINA_Plugin.csproj
│   │   ├── TargetFramework: net472 (or net6.0 for newer NINA)
│   │   ├── Dependencies:
│   │   │   ├── NINA.Core
│   │   │   ├── NINA.Equipment
│   │   │   ├── Newtonsoft.Json
│   │   │   └── System.Reactive
│   │   └── Build output: bin/Release/*.dll
│   │
│   ├── Commands/
│   │   ├── SeeStar_TakeExposureCommand.cs
│   │   ├── SeeStar_AutofocusCommand.cs
│   │   └── SeeStar_StopExposureCommand.cs
│   │
│   ├── UI/
│   │   ├── AutofocusTab.xaml
│   │   ├── AutofocusTab.xaml.cs
│   │   ├── AutofocusTabViewModel.cs
│   │   ├── MessageDisplayPanel.xaml
│   │   ├── MessageDisplayPanel.xaml.cs
│   │   ├── SettingsView.xaml
│   │   └── SettingsView.xaml.cs
│   │
│   ├── Services/
│   │   ├── SeeStar_APIClient.cs
│   │   ├── StackingService.cs
│   │   ├── AutofocusService.cs
│   │   ├── MessagePollingService.cs
│   │   ├── StateManager.cs
│   │   └── ConfigurationService.cs
│   │
│   ├── Models/
│   │   ├── ExposureRequest.cs
│   │   ├── ExposureResponse.cs
│   │   ├── AutofocusRequest.cs
│   │   ├── AutofocusResponse.cs
│   │   ├── StackStatus.cs
│   │   ├── SeeStar_Message.cs
│   │   ├── ApiResponse.cs
│   │   └── FocusPosition.cs
│   │
│   ├── Utilities/
│   │   ├── Logger.cs
│   │   ├── Constants.cs
│   │   ├── Extensions.cs
│   │   └── Converters.xaml.cs (for binding)
│   │
│   ├── Events/
│   │   ├── StackStatusChangedEventArgs.cs
│   │   ├── FocusPositionChangedEventArgs.cs
│   │   ├── AutofocusStatusChangedEventArgs.cs
│   │   └── MessageReceivedEventArgs.cs
│   │
│   ├── Plugin/
│   │   ├── PluginModule.cs (entry point)
│   │   └── PluginSettings.cs
│   │
│   └── Properties/
│       └── AssemblyInfo.cs
│
├── tests/
│   ├── SeeStar_NINA_Plugin.Tests.csproj
│   ├── CommandTests/
│   │   ├── ExposureCommandTests.cs
│   │   └── AutofocusCommandTests.cs
│   ├── ServiceTests/
│   │   ├── APIClientTests.cs
│   │   ├── StackingServiceTests.cs
│   │   └── StateManagerTests.cs
│   └── Fixtures/
│       └── MockSeeStar_APIClient.cs
│
├── .github/workflows/
│   ├── build.yml (CI/CD)
│   ├── test.yml
│   └── release.yml
│
├── SeeStar_NINA_Plugin.nuspec
├── README.md
├── CHANGELOG.md
└── LICENSE.md
```

---

## Detailed Class Implementations

### 1. SeeStar_APIClient.cs

```csharp
using System;
using System.Net.Sockets;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using NINA.Core.Utility;

namespace SeeStar_NINA_Plugin.Services
{
    public class SeeStar_APIClient
    {
        private readonly string _host;
        private readonly int _port;
        private readonly ILogger _logger;
        
        private TcpClient _tcpClient;
        private NetworkStream _networkStream;
        private int _requestId = 0;
        
        private const int REQUEST_TIMEOUT_MS = 10000;
        private const int BUFFER_SIZE = 8192;

        public SeeStar_APIClient(string host, int port, int deviceNumber, ILogger logger)
        {
            _host = host;
            _port = port; // 4700 for imaging, 4800 for metadata
            _logger = logger;
        }

        /// <summary>
        /// Establish TCP connection to SeeStar S50
        /// </summary>
        public async Task<bool> ConnectAsync()
        {
            try
            {
                _tcpClient = new TcpClient();
                await _tcpClient.ConnectAsync(_host, _port);
                _networkStream = _tcpClient.GetStream();
                
                _logger.Info($"Connected to SeeStar at {_host}:{_port}");
                return true;
            }
            catch (Exception ex)
            {
                _logger.Error($"Failed to connect to SeeStar: {ex.Message}");
                return false;
            }
        }

        /// <summary>
        /// Send a method call to SeeStar via TCP Socket and return the response
        /// Protocol: Send JSON command followed by \r\n, read JSON response
        /// </summary>
        public async Task<ApiResponse> SendMethodSyncAsync(string method, object parameters = null)
        {
            if (_networkStream == null || !_tcpClient.Connected)
            {
                throw new SeeStar_APIException("Not connected to SeeStar. Call ConnectAsync first.");
            }

            try
            {
                _requestId++;
                
                // Build JSON command object
                var commandObject = new
                {
                    id = _requestId,
                    method = method,
                    @params = parameters ?? new object() { }
                };

                string commandJson = JsonConvert.SerializeObject(commandObject);
                byte[] commandBytes = Encoding.UTF8.GetBytes(commandJson + "\r\n");

                _logger.Debug($"SeeStar TCP: Sending {method} (ID: {_requestId})");
                
                // Send command
                await _networkStream.WriteAsync(commandBytes, 0, commandBytes.Length);
                await _networkStream.FlushAsync();

                // Read response with timeout
                return await ReadResponseAsync(REQUEST_TIMEOUT_MS);
            }
            catch (IOException ex)
            {
                _logger.Error($"TCP I/O error calling {method}: {ex.Message}");
                throw new SeeStar_APIException($"Connection error: {ex.Message}", ex);
            }
            catch (Exception ex)
            {
                _logger.Error($"Error calling {method}: {ex.Message}");
                throw new SeeStar_APIException($"Protocol error: {ex.Message}", ex);
            }
        }

        /// <summary>
        /// Read JSON response from TCP stream with timeout
        /// </summary>
        private async Task<ApiResponse> ReadResponseAsync(int timeoutMs)
        {
            var cts = new CancellationTokenSource(timeoutMs);
            byte[] buffer = new byte[BUFFER_SIZE];
            int bytesRead = 0;

            try
            {
                // Read from stream until we get a complete JSON line (ending with \n or \r\n)
                StringBuilder responseBuilder = new StringBuilder();
                
                while (true)
                {
                    bytesRead = await _networkStream.ReadAsync(buffer, 0, BUFFER_SIZE, cts.Token);
                    
                    if (bytesRead == 0)
                    {
                        throw new SeeStar_APIException("Connection closed by SeeStar");
                    }

                    string chunk = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                    responseBuilder.Append(chunk);

                    // Check if we have a complete line
                    string response = responseBuilder.ToString();
                    if (response.Contains("\n") || response.Contains("\r\n"))
                    {
                        // Extract first complete line
                        int lineEnd = response.IndexOfAny(new[] { '\n', '\r' });
                        string jsonLine = response.Substring(0, lineEnd).Trim();

                        if (!string.IsNullOrEmpty(jsonLine))
                        {
                            _logger.Debug($"SeeStar TCP: Response received: {jsonLine.Substring(0, Math.Min(100, jsonLine.Length))}...");

                            try
                            {
                                var jsonResponse = JObject.Parse(jsonLine);
                                
                                // Check for error in response
                                if (jsonResponse["error"] != null)
                                {
                                    string errorMsg = jsonResponse["error"].ToString();
                                    _logger.Warn($"SeeStar error: {errorMsg}");
                                    throw new SeeStar_APIException($"SeeStar error: {errorMsg}");
                                }

                                // Build ApiResponse object
                                var apiResp = new ApiResponse
                                {
                                    Value = new ApiResponseValue
                                    {
                                        Result = jsonResponse["result"],
                                        Error = null
                                    }
                                };

                                return apiResp;
                            }
                            catch (JsonException ex)
                            {
                                _logger.Error($"Failed to parse JSON response: {ex.Message}");
                                throw new SeeStar_APIException($"Invalid response format: {ex.Message}");
                            }
                        }
                    }
                }
            }
            catch (OperationCanceledException)
            {
                _logger.Error("TCP response timeout");
                throw new SeeStar_APIException("Request timeout", new TimeoutException());
            }
        }

        public async Task<bool> StartStackAsync(int gain, bool restart)
        {
            var result = await SendMethodSyncAsync(
                "iscope_start_stack",
                new { restart }
            );
            
            if (result?.Value?.Error != null)
                return false;

            // Set gain
            await SendMethodSyncAsync(
                "set_control_value",
                new object[] { "gain", gain }
            );

            return true;
        }

        public async Task<bool> StopStackAsync()
        {
            var result = await SendMethodSyncAsync(
                "iscope_stop_view",
                new { stage = "Stack" }
            );

            return result?.Value?.Error == null;
        }

        public async Task<StackStatus> GetStackStatusAsync()
        {
            var result = await SendMethodSyncAsync("get_camera_state");
            
            if (result?.Value?.Result is StackStatus status)
                return status;

            return null;
        }

        public async Task<bool> StartAutofocusAsync()
        {
            var result = await SendMethodSyncAsync("start_auto_focuse");
            return result?.Value?.Error == null;
        }

        public async Task<bool> StopAutofocusAsync()
        {
            var result = await SendMethodSyncAsync("stop_auto_focuse");
            return result?.Value?.Error == null;
        }

        public async Task<FocusPosition> GetFocusPositionAsync()
        {
            var result = await SendMethodSyncAsync(
                "get_focuser_position",
                new { ret_obj = true }
            );

            if (result?.Value?.Result is FocusPosition pos)
                return pos;

            return null;
        }

        public async Task<StackStatus> GetEventStateAsync()
        {
            var result = await SendMethodSyncAsync("get_event_state");
            return result?.Value?.Result as StackStatus;
        }

        private int GenerateTransactionId() => DateTime.Now.Millisecond;

        public void Disconnect()
        {
            try
            {
                if (_networkStream != null)
                {
                    _networkStream.Close();
                    _networkStream.Dispose();
                }

                if (_tcpClient != null)
                {
                    _tcpClient.Close();
                    _tcpClient.Dispose();
                }

                _logger.Info("Disconnected from SeeStar");
            }
            catch (Exception ex)
            {
                _logger.Error($"Error disconnecting from SeeStar: {ex.Message}");
            }
        }

        public void Dispose()
        {
            Disconnect();
        }
    }

    public class SeeStar_APIException : Exception
    {
        public SeeStar_APIException(string message) : base(message) { }
        public SeeStar_APIException(string message, Exception innerException) 
            : base(message, innerException) { }
    }
}
```

---

### 2. StackingService.cs

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using NINA.Core.Utility;

namespace SeeStar_NINA_Plugin.Services
{
    public class StackingService
    {
        private readonly SeeStar_APIClient _apiClient;
        private readonly ILogger _logger;
        
        private CancellationTokenSource _cancellationToken;
        private DateTime _externalStartTime;
        private double _totalDurationSeconds;
        private bool _isStacking = false;

        public event EventHandler<StackStatusChangedEventArgs> StatusChanged;

        public StackingService(SeeStar_APIClient apiClient, ILogger logger)
        {
            _apiClient = apiClient;
            _logger = logger;
        }

        public async Task<ExposureResponse> StartExposureAsync(ExposureRequest request)
        {
            var response = new ExposureResponse();

            try
            {
                _logger.Info($"Starting exposure: {request.TotalDurationSeconds}s @ {request.Gain} gain");

                // Start stack
                var started = await _apiClient.StartStackAsync(request.Gain, request.Restart);
                if (!started)
                {
                    response.Success = false;
                    response.Error = "Failed to start stack on SeeStar";
                    return response;
                }

                response.SequenceId = Guid.NewGuid().ToString();
                response.Success = true;
                response.StatusMessage = "Stack started - acquiring frames";

                _totalDurationSeconds = request.TotalDurationSeconds;
                _externalStartTime = DateTime.UtcNow;
                _isStacking = true;

                // Start polling thread (non-blocking)
                _ = PollStackStatusAsync(request.TotalDurationSeconds);

                return response;
            }
            catch (Exception ex)
            {
                _logger.Error($"Error starting exposure: {ex}");
                response.Success = false;
                response.Error = ex.Message;
                return response;
            }
        }

        public async Task<bool> StopExposureAsync()
        {
            _logger.Info("Stopping exposure");
            _isStacking = false;
            _cancellationToken?.Cancel();

            return await _apiClient.StopStackAsync();
        }

        private async Task PollStackStatusAsync(double durationSeconds)
        {
            _cancellationToken = new CancellationTokenSource();
            var deadline = DateTime.UtcNow.AddSeconds(durationSeconds);
            
            const int POLL_INTERVAL_MS = 500;

            try
            {
                while (!_cancellationToken.Token.IsCancellationRequested && _isStacking)
                {
                    if (DateTime.UtcNow > deadline)
                    {
                        _logger.Info("Exposure duration complete - stopping stack");
                        await StopExposureAsync();
                        break;
                    }

                    try
                    {
                        var status = await _apiClient.GetStackStatusAsync();
                        if (status != null)
                        {
                            var elapsed = (DateTime.UtcNow - _externalStartTime).TotalSeconds;
                            StatusChanged?.Invoke(this, new StackStatusChangedEventArgs
                            {
                                FrameCount = status.FrameCount,
                                RejectedCount = status.RejectedCount,
                                ElapsedSeconds = elapsed,
                                TotalSeconds = durationSeconds,
                                SnrMetric = status.SnrMetric
                            });
                        }
                    }
                    catch (SeeStar_APIException ex)
                    {
                        _logger.Warn($"Error polling stack status: {ex.Message}");
                    }

                    await Task.Delay(POLL_INTERVAL_MS, _cancellationToken.Token);
                }
            }
            finally
            {
                _isStacking = false;
                _cancellationToken?.Dispose();
            }
        }
    }

    public class StackStatusChangedEventArgs : EventArgs
    {
        public int FrameCount { get; set; }
        public int RejectedCount { get; set; }
        public double ElapsedSeconds { get; set; }
        public double TotalSeconds { get; set; }
        public double SnrMetric { get; set; }
    }
}
```

---

### 3. AutofocusService.cs

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using NINA.Core.Utility;

namespace SeeStar_NINA_Plugin.Services
{
    public class AutofocusService
    {
        private readonly SeeStar_APIClient _apiClient;
        private readonly ILogger _logger;
        
        private CancellationTokenSource _cancellationToken;
        private bool _isFocusing = false;

        public event EventHandler<FocusPositionChangedEventArgs> PositionChanged;
        public event EventHandler<AutofocusStatusChangedEventArgs> StatusChanged;

        public AutofocusService(SeeStar_APIClient apiClient, ILogger logger)
        {
            _apiClient = apiClient;
            _logger = logger;
        }

        public async Task<AutofocusResponse> ExecuteAutofocusAsync(AutofocusRequest request)
        {
            var response = new AutofocusResponse();

            try
            {
                var startTime = DateTime.UtcNow;
                int attempt = 1;

                for (attempt = 1; attempt <= request.TryCount; attempt++)
                {
                    _logger.Info($"Autofocus attempt {attempt} of {request.TryCount}");

                    // Start autofocus
                    var started = await _apiClient.StartAutofocusAsync();
                    if (!started)
                    {
                        response.Attempt = attempt;
                        response.Error = "Failed to start autofocus";
                        continue;
                    }

                    _isFocusing = true;
                    StatusChanged?.Invoke(this, new AutofocusStatusChangedEventArgs 
                    { 
                        Status = "Focusing...", 
                        Attempt = attempt, 
                        TotalAttempts = request.TryCount 
                    });

                    // Poll for completion
                    var completed = await PollAutofocusAsync(request.TimeoutSeconds);
                    
                    if (completed)
                    {
                        var position = await _apiClient.GetFocusPositionAsync();
                        var elapsed = (DateTime.UtcNow - startTime).TotalSeconds;

                        response.Success = true;
                        response.FinalPosition = position?.Position ?? -1;
                        response.Attempt = attempt;
                        response.ElapsedTimeSeconds = elapsed;
                        response.StatusMessage = "Focus complete";

                        _isFocusing = false;
                        return response;
                    }

                    _isFocusing = false;

                    if (attempt < request.TryCount)
                    {
                        await Task.Delay(5000); // Wait 5s between attempts
                    }
                }

                response.Success = false;
                response.Attempt = attempt;
                response.Error = "Autofocus failed after all attempts";
                return response;
            }
            catch (Exception ex)
            {
                _logger.Error($"Error executing autofocus: {ex}");
                response.Success = false;
                response.Error = ex.Message;
                return response;
            }
        }

        private async Task<bool> PollAutofocusAsync(int timeoutSeconds)
        {
            _cancellationToken = new CancellationTokenSource();
            var deadline = DateTime.UtcNow.AddSeconds(timeoutSeconds);
            
            const int POLL_INTERVAL_MS = 200;

            try
            {
                while (!_cancellationToken.Token.IsCancellationRequested && _isFocusing)
                {
                    if (DateTime.UtcNow > deadline)
                    {
                        _logger.Warn("Autofocus timeout");
                        return false;
                    }

                    try
                    {
                        // Get current position
                        var position = await _apiClient.GetFocusPositionAsync();
                        if (position != null)
                        {
                            PositionChanged?.Invoke(this, new FocusPositionChangedEventArgs
                            {
                                Position = position.Position
                            });
                        }

                        // Check autofocus state
                        var state = await _apiClient.GetEventStateAsync();
                        if (state?.AutoFocusState == "complete")
                        {
                            _logger.Info("Autofocus complete");
                            return true;
                        }
                    }
                    catch (SeeStar_APIException ex)
                    {
                        _logger.Warn($"Error polling autofocus: {ex.Message}");
                    }

                    await Task.Delay(POLL_INTERVAL_MS, _cancellationToken.Token);
                }

                return false;
            }
            finally
            {
                _cancellationToken?.Dispose();
            }
        }
    }

    public class FocusPositionChangedEventArgs : EventArgs
    {
        public int Position { get; set; }
    }

    public class AutofocusStatusChangedEventArgs : EventArgs
    {
        public string Status { get; set; }
        public int Attempt { get; set; }
        public int TotalAttempts { get; set; }
    }
}
```

---

### 4. SeeStar_TakeExposureCommand.cs (NINA Command)

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using NINA.Core.Model;
using NINA.Equipment.Interfaces;
using NINA.Plugin;
using NINA.Plugin.Interfaces;

namespace SeeStar_NINA_Plugin.Commands
{
    public class SeeStar_TakeExposureCommand : PluginCommand
    {
        private readonly StackingService _stackingService;
        
        public override string Name => "SeeStar_Take_Exposure";
        public override string Description => "Start a SeeStar imaging sequence";
        public override string Category => "SeeStar";

        public override string InputType => typeof(ExposureRequest).Name;
        public override string OutputType => typeof(ExposureResponse).Name;

        public SeeStar_TakeExposureCommand(StackingService stackingService)
        {
            _stackingService = stackingService;
        }

        public override async Task<string> Execute(
            string input, 
            CancellationToken ct,
            IProgress<ApplicationStatus> progress)
        {
            try
            {
                // Parse input JSON
                var request = JsonConvert.DeserializeObject<ExposureRequest>(input);

                progress?.Report(new ApplicationStatus 
                { 
                    Status = "Starting SeeStar exposure..." 
                });

                // Execute exposure
                var response = await _stackingService.StartExposureAsync(request);

                // Return response as JSON
                return JsonConvert.SerializeObject(response);
            }
            catch (Exception ex)
            {
                return JsonConvert.SerializeObject(new ExposureResponse
                {
                    Success = false,
                    Error = ex.Message
                });
            }
        }
    }
}
```

---

### 5. AutofocusTab.xaml

```xaml
<UserControl x:Class="SeeStar_NINA_Plugin.UI.AutofocusTab"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    Background="{DynamicResource BackgroundBrush}"
    Foreground="{DynamicResource PrimaryTextBrush}">
    
    <Grid Margin="8">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <!-- Focus Position Display -->
        <StackPanel Grid.Row="0" Margin="0,0,0,8">
            <TextBlock Text="Focus Position:" FontWeight="Bold"/>
            <TextBlock Text="{Binding FocusPosition, StringFormat=Position: {0}}" 
                       FontSize="16" 
                       Foreground="{DynamicResource AccentBrush3}"/>
        </StackPanel>

        <!-- Status Display-->
        <StackPanel Grid.Row="1" Margin="0,0,0,8">
            <TextBlock Text="Status:" FontWeight="Bold"/>
            <TextBlock Text="{Binding Status}" 
                       FontSize="14" 
                       Foreground="{DynamicResource WarningBrush}"/>
        </StackPanel>

        <!-- Retry Selection & Buttons -->
        <StackPanel Grid.Row="2" Margin="0,0,0,8">
            <StackPanel Orientation="Horizontal" Margin="0,0,0,8">
                <Label Content="Retry Attempts:" VerticalAlignment="Center"/>
                <ComboBox ItemsSource="{Binding RetryOptions}" 
                          SelectedItem="{Binding SelectedRetryCount}"
                          Width="60"/>
            </StackPanel>

            <StackPanel Orientation="Horizontal" Spacing="8">
                <Button Content="Run Autofocus" 
                        Command="{Binding RunAutofocusCommand}"
                        IsEnabled="{Binding IsNotRunning}"
                        Padding="12,6"
                        Background="{DynamicResource AccentBrush}"/>
                
                <Button Content="Stop Autofocus" 
                        Command="{Binding StopAutofocusCommand}"
                        IsEnabled="{Binding IsRunning}"
                        Padding="12,6"
                        Background="{DynamicResource WarningBrush}"/>
            </StackPanel>

            <!-- Progress Bar -->
            <ProgressBar Value="{Binding Progress}" 
                        Maximum="100"
                        Height="20"
                        Margin="0,8,0,0"/>
        </StackPanel>

        <!-- Message Log -->
        <StackPanel Grid.Row="3" Margin="0,8,0,0">
            <TextBlock Text="Recent Messages" FontWeight="Bold" Margin="0,0,0,4"/>
            <ListBox ItemsSource="{Binding MessageLog}"
                    Height="200"
                    ScrollViewer.VerticalScrollBarVisibility="Auto"
                    Background="{DynamicResource BackgroundBrush2}">
                <ListBox.ItemTemplate>
                    <DataTemplate>
                        <TextBlock Text="{Binding}" 
                                  TextWrapping="Wrap"
                                  Margin="4,2"/>
                    </DataTemplate>
                </ListBox.ItemTemplate>
            </ListBox>
        </StackPanel>
    </Grid>
</UserControl>
```

---

### 6. AutofocusTabViewModel.cs

```csharp
using System;
using System.Collections.ObjectModel;
using System.Windows.Input;
using NINA.Core.Utility;

namespace SeeStar_NINA_Plugin.UI
{
    public class AutofocusTabViewModel : NotifyPropertyChanged
    {
        private readonly AutofocusService _autofocusService;
        private readonly ILogger _logger;

        private int _focusPosition = -1;
        private string _status = "Ready";
        private int _selectedRetryCount = 1;
        private bool _isRunning = false;
        private int _progress = 0;

        public ObservableCollection<string> MessageLog { get; }
        public ObservableCollection<int> RetryOptions { get; }

        public int FocusPosition
        {
            get => _focusPosition;
            set 
            { 
                if (value != _focusPosition)
                {
                    _focusPosition = value;
                    RaisePropertyChanged();
                }
            }
        }

        public string Status
        {
            get => _status;
            set
            {
                if (value != _status)
                {
                    _status = value;
                    RaisePropertyChanged();
                }
            }
        }

        public int SelectedRetryCount
        {
            get => _selectedRetryCount;
            set => RaiseAndSetIfChanged(ref _selectedRetryCount, value);
        }

        public bool IsRunning
        {
            get => _isRunning;
            set => RaiseAndSetIfChanged(ref _isRunning, value);
        }

        public bool IsNotRunning => !IsRunning;

        public int Progress
        {
            get => _progress;
            set => RaiseAndSetIfChanged(ref _progress, value);
        }

        public ICommand RunAutofocusCommand { get; }
        public ICommand StopAutofocusCommand { get; }

        public AutofocusTabViewModel(AutofocusService autofocusService, ILogger logger)
        {
            _autofocusService = autofocusService;
            _logger = logger;

            MessageLog = new ObservableCollection<string>();
            RetryOptions = new ObservableCollection<int> { 1, 2, 3, 4, 5 };

            RunAutofocusCommand = new RelayCommand(RunAutofocus, IsNotRunning);
            StopAutofocusCommand = new RelayCommand(StopAutofocus, () => IsRunning);

            // Subscribe to service events
            _autofocusService.PositionChanged += OnPositionChanged;
            _autofocusService.StatusChanged += OnStatusChanged;
        }

        private async void RunAutofocus()
        {
            IsRunning = true;
            Status = "Starting autofocus...";
            Progress = 0;
            MessageLog.Clear();

            var request = new AutofocusRequest
            {
                TryCount = SelectedRetryCount,
                TimeoutSeconds = 120
            };

            var response = await _autofocusService.ExecuteAutofocusAsync(request);

            if (response.Success)
            {
                Status = $"Complete - Position: {response.FinalPosition}";
                AddMessage($"✓ Autofocus complete. Position: {response.FinalPosition}");
                Progress = 100;
            }
            else
            {
                Status = $"Failed - {response.Error}";
                AddMessage($"✗ Autofocus failed: {response.Error}");
            }

            IsRunning = false;
        }

        private async void StopAutofocus()
        {
            Status = "Stopping...";
            await _autofocusService.StopFocusingAsync();
            Status = "Stopped";
            IsRunning = false;
        }

        private void OnPositionChanged(object sender, FocusPositionChangedEventArgs e)
        {
            FocusPosition = e.Position;
        }

        private void OnStatusChanged(object sender, AutofocusStatusChangedEventArgs e)
        {
            Status = e.Status;
            Progress = (e.Attempt * 100) / e.TotalAttempts;
            AddMessage($"Attempt {e.Attempt}/{e.TotalAttempts}: {e.Status}");
        }

        private void AddMessage(string message)
        {
            Application.Current?.Dispatcher?.Invoke(() =>
            {
                var timestamped = $"[{DateTime.Now:HH:mm:ss}] {message}";
                MessageLog.Insert(0, timestamped);

                // Keep only last 10
                while (MessageLog.Count > 10)
                    MessageLog.RemoveAt(MessageLog.Count - 1);
            });
        }

        public void Dispose()
        {
            if (_autofocusService != null)
            {
                _autofocusService.PositionChanged -= OnPositionChanged;
                _autofocusService.StatusChanged -= OnStatusChanged;
            }
        }
    }
}
```

---

## Models

### ExposureRequest.cs
```csharp
public enum ExposureType { LIGHT, DARK }

public class ExposureRequest
{
    public double TotalDurationSeconds { get; set; }
    public double SubExposureTimeSeconds { get; set; }
    public ExposureType Type { get; set; }
    public int Gain { get; set; }
    public bool Restart { get; set; } = true;
}

public class ExposureResponse
{
    public string SequenceId { get; set; }
    public bool Success { get; set; }
    public int FinalFrameCount { get; set; }
    public int RejectedCount { get; set; }
    public string StatusMessage { get; set; }
    public string Error { get; set; }
}
```

### AutofocusRequest.cs
```csharp
public class AutofocusRequest
{
    public int TryCount { get; set; }
    public int TimeoutSeconds { get; set; }
}

public class AutofocusResponse
{
    public bool Success { get; set; }
    public int FinalPosition { get; set; }
    public int Attempt { get; set; }
    public double ElapsedTimeSeconds { get; set; }
    public string StatusMessage { get; set; }
    public string Error { get; set; }
}
```

### ApiResponse.cs
```csharp
public class ApiResponse
{
    [JsonProperty("Value")]
    public ApiResponseValue Value { get; set; }
}

public class ApiResponseValue
{
    [JsonProperty("Result")]
    public object Result { get; set; }
    
    [JsonProperty("Error")]
    public string Error { get; set; }
}
```

---

## Unit Test Example

### StackingServiceTests.cs

```csharp
using Xunit;
using Moq;

namespace SeeStar_NINA_Plugin.Tests
{
    public class StackingServiceTests
    {
        [Fact]
        public async Task StartExposure_WithValidRequest_ReturnsSuccess()
        {
            // Arrange
            var mockClient = new Mock<SeeStar_APIClient>();
            mockClient.Setup(c => c.StartStackAsync(It.IsAny<int>(), It.IsAny<bool>()))
                .ReturnsAsync(true);

            var mockLogger = new Mock<ILogger>();
            var service = new StackingService(mockClient.Object, mockLogger.Object);

            var request = new ExposureRequest
            {
                TotalDurationSeconds = 120,
                SubExposureTimeSeconds = 2,
                Type = ExposureType.LIGHT,
                Gain = 75,
                Restart = true
            };

            // Act
            var response = await service.StartExposureAsync(request);

            // Assert
            Assert.True(response.Success);
            Assert.NotNull(response.SequenceId);
            Assert.Null(response.Error);
        }

        [Fact]
        public async Task StopExposure_WhenRunning_StopsStack()
        {
            // Arrange
            var mockClient = new Mock<SeeStar_APIClient>();
            mockClient.Setup(c => c.StopStackAsync())
                .ReturnsAsync(true);

            var mockLogger = new Mock<ILogger>();
            var service = new StackingService(mockClient.Object, mockLogger.Object);

            // Act
            var result = await service.StopExposureAsync();

            // Assert
            Assert.True(result);
            mockClient.Verify(c => c.StopStackAsync(), Times.Once);
        }
    }
}
```

---

## Build Configuration

### SeeStar_NINA_Plugin.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk.WindowsDesktop">
  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
    <UseWPF>true</UseWPF>
    <AssemblyVersion>1.0.0</AssemblyVersion>
    <FileVersion>1.0.0</FileVersion>
    <Version>1.0.0</Version>
    <Authors>Your Name</Authors>
    <Description>SeeStar S50 Control Plugin for N.I.N.A</Description>
    <DebugType>embedded</DebugType>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="NINA.Core" Version="[3.0,4.0)" />
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="System.Reactive" Version="5.4.1" />
  </ItemGroup>

  <ItemGroup Condition="'$(Configuration)'=='Debug'">
    <PackageReference Include="xunit" Version="2.6.0" />
    <PackageReference Include="Moq" Version="4.18.4" />
  </ItemGroup>
</Project>
```

---

## CI/CD Pipeline

### .github/workflows/build.yml
```yaml
name: Build

on: [push, pull_request]

jobs:
  build:
    runs-on: windows-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: '6.0.x'
    
    - name: Restore
      run: dotnet restore src/SeeStar_NINA_Plugin.csproj
    
    - name: Build
      run: dotnet build src/SeeStar_NINA_Plugin.csproj --configuration Release
    
    - name: Test
      run: dotnet test tests/SeeStar_NINA_Plugin.Tests.csproj
    
    - name: Pack NuGet
      run: dotnet pack src/SeeStar_NINA_Plugin.csproj --no-build --configuration Release --output nupkg
    
    - name: Upload Artifacts
      uses: actions/upload-artifact@v2
      with:
        name: nupkg
        path: nupkg/
```

---

## Next Steps for Development

1. **Create project from this structure**
   - Follow directory layout above
   - Copy model classes
   - Setup project file

2. **Implement Phase 1 (API Client)**
   - Finish `SeeStar_APIClient.cs`
   - Add unit tests
   - Verify HTTP communication with mock

3. **Implement Services**
   - Complete `StackingService.cs`
   - Complete `AutofocusService.cs`
   - Add state management

4. **Implement Commands**
   - Register with NINA
   - Test in sequencer

5. **Build UI**
   - Create XAML files
   - Implement ViewModels
   - Test integration

---

**Document Version:** 1.0  
**Status:** Ready for development
