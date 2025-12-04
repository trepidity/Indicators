# ATAS Indicators: AI Coding Agent Instructions

## Architecture Overview

This is the **ATAS Platform Technical Indicators** library—a C# .NET 8 component system providing 200+ trading indicators for the ATAS trading platform. The architecture is built around extensible base classes and composition patterns.

**Key Dependencies:**
- External ATAS libraries (via project references): `Indicators`, `Rendering`, `Localization`, `Common`, `Attributes`, `DataFeedsCore`
- `Newtonsoft.Json` for serialization
- Cross-platform support (Windows/WPF and cross-platform with Avalonia)

**Build Configurations:**
- **Platforms:** `AnyCPU` (Windows + WPF), `Cross` (cross-platform, no WPF)
- **Configurations:** `Debug`, `Release`, `Publish`
- Assembly signing enabled; key file at `../../../GitLab/OFT/OFT.snk`

## Indicator Implementation Pattern

All indicators inherit from the `Indicator` base class and follow a strict convention:

### Required Structure (Example: `EMA.cs`)
```csharp
namespace ATAS.Indicators.Technical;

[DisplayName("Display Name")]
[Display(ResourceType = typeof(Strings), Description = nameof(Strings.DescriptionKey))]
[HelpLink("https://help.atas.net/support/...")]
public class IndicatorName : Indicator
{
    #region Fields
    private int _period = 10;
    private ValueDataSeries _renderSeries = new("RenderSeries", "Label");
    #endregion

    #region Properties
    [Parameter]
    [Display(ResourceType = typeof(Strings), Name = nameof(Strings.Period), 
        GroupName = nameof(Strings.Settings), Description = nameof(Strings.PeriodDescription), Order = 20)]
    [Range(1, 10000)]
    public int Period
    {
        get => _period;
        set
        {
            if (_period == value) return;
            _period = value;
            RaisePropertyChanged(nameof(Period));
            RecalculateValues();
        }
    }
    #endregion

    #region ctor
    public IndicatorName() : base(true)  // true = new panel
    {
        Panel = IndicatorDataProvider.NewPanel;
        DataSeries[0] = _renderSeries;
        // Add child indicators
        Add(childIndicator);
    }
    #endregion

    #region Protected methods
    protected override void OnCalculate(int bar, decimal value)
    {
        _renderSeries[bar] = CalculateValue(bar);
    }
    #endregion
}
```

### Key Conventions

1. **Namespace:** Always `ATAS.Indicators.Technical` or `ATAS.Indicators.Technical;` (file-scoped)
2. **Attributes:** Required `[DisplayName]`, `[Display]` with localization strings, `[HelpLink]`
3. **Data Series:**
   - Use `ValueDataSeries` for single-value lines
   - Use `RangeDataSeries` for bands/channels
   - Use `BarDataSeries` for OHLC data
   - Use `ObjectDataSeries` for generic objects
4. **Parameters:** Mark settings with `[Parameter]` + `[Display]` + `[Range]`
5. **Constructor:** Call `base(true)` for new panel, assign `DataSeries[0]` or replace with custom series
6. **Calculation:** Override `OnCalculate(int bar, decimal value)` — called once per bar
7. **Access candles:** Use `GetCandle(bar)` to access OHLC data for current/previous bars

## Composition Pattern for Complex Indicators

Use composition (not inheritance) for indicators built on other indicators:

```csharp
public class CompositeIndicator : Indicator
{
    private readonly SimpleIndicator _baseIndicator = new();
    private readonly ValueDataSeries _outputSeries = new("Output", "Label");

    public CompositeIndicator() : base(true)
    {
        Add(_baseIndicator);  // Register child for lifecycle management
        DataSeries[0] = _outputSeries;
    }

    protected override void OnCalculate(int bar, decimal value)
    {
        var baseValue = ((ValueDataSeries)_baseIndicator.DataSeries[0])[bar];
        _outputSeries[bar] = Transform(baseValue);
    }
}
```

**Rules:**
- Call `Add(childIndicator)` to register children → ensures proper initialization and recalculation
- Access child results via `_child.DataSeries[n][bar]` cast appropriately
- When updating child parameters, always call `RecalculateValues()` on parent
- Parameters on child indicators are automatically exposed as parent properties

## Visualization & Rendering

For indicators with custom drawing (`OnRender`):

```csharp
protected override void OnRender(RenderContext context, DrawingLayouts layout)
{
    // Access chart drawing primitives via context
    // Called after OnCalculate for rendering phase
    // Use layout to determine positioning
}
```

Access price data via `GetCandle(bar)` (returns `Candle` with `High`, `Low`, `Open`, `Close`, `Volume`).

## Localization

All UI strings come from `Strings` resource class (via `OFT.Localization`):
- `Strings.Period` → "Period"
- `Strings.PeriodDescription` → longer description
- Lookup keys in existing indicators for naming conventions
- Keep descriptions technical but user-friendly

## Cross-Platform Support

The codebase supports two platform builds:

| Constant | Platform | Dependencies |
|----------|----------|--------------|
| `CROSS_PLATFORM` (defined for "Cross" platform) | Avalonia (cross-platform UI) | No WPF, uses `System.Drawing` |
| (not defined) | Windows/WPF | Uses `System.Windows.Media` |

**Platform-specific code example** (from `GlobalUsings.cs`):
```csharp
#if CROSS_PLATFORM
    global using CrossColor = System.Drawing.Color;
    global using CrossKey = Avalonia.Input.Key;
#else
    global using CrossColor = System.Windows.Media.Color;
    global using CrossKey = System.Windows.Input.Key;
#endif
```

**Guidance:** Use `CrossColor` and `CrossPen` aliases for platform-agnostic code.

## Common Patterns

### Adding Line Series
```csharp
LineSeries _line80;

public LineSeries Line80
{
    get => _line80;
    set => _line80 = value;
}

// In constructor:
LineSeries.Add(_line80);
```

### Multiple Output Series
```csharp
public Indicator() : base(true)
{
    DataSeries[0] = _series1;
    DataSeries.Add(_series2);
    DataSeries.Add(_series3);
}
```

### Minimized Mode (Small Values Display)
```csharp
DataSeries[0].UseMinimizedModeIfEnabled = true;
```

### Alert/Signal Management
```csharp
private int _lastAlert;

protected override void OnCalculate(int bar, decimal value)
{
    if (crossoverCondition && bar != _lastAlert)
    {
        AddAlert("CrossoverAlert", "Description");
        _lastAlert = bar;
    }
}
```

## File Organization

```
Technical/
├── Core indicator files (*.cs) - one per indicator class
├── ClusterSearch.cs / ClusterSearch.Models.cs / ClusterSearch.SeriesHandling.cs (for complex indicators)
├── GlobalUsings.cs (platform abstraction definitions)
├── Indicators.Technical.csproj
└── Properties/ (resources, assembly info)
```

Each indicator = one file (one public class).

## Build & Development

**Building from command line:**
```bash
dotnet build Indicators.sln -c Release -p Platform=AnyCPU
dotnet build Indicators.sln -c Release -p Platform=Cross  # Cross-platform
```

**Known Build Issues:**
- Temporary `GenerateDepsFile` conflicts resolved by waiting and retrying
- Platform-specific compilation: `AnyCPU` includes WPF; `Cross` excludes it

## Key Files to Reference

- **Base class behavior:** Examine `EMA.cs` (simple) or `BollingerBands.cs` (multi-series)
- **Composition pattern:** `AroonOscillator.cs`, `ChaikinOscillator.cs`
- **Rendering:** `OHLCPlus.cs`, `DailyLines.cs`
- **Cross-platform code:** `GlobalUsings.cs`
- **Complex logic:** `ClusterStatistic.cs`, `VWAP.cs`

## Property Change Pattern

Use `RaisePropertyChanged()` to notify UI of parameter changes:
```csharp
public int Period
{
    get => _period;
    set
    {
        if (_period == value) return;
        _period = value;
        RaisePropertyChanged(nameof(Period));
        RecalculateValues();
    }
}
```

**Always:**
- Guard against no-op updates (`if (oldValue == newValue) return`)
- Call `RecalculateValues()` for changes affecting calculations
- Call `RaisePropertyChanged()` for UI updates

## External Documentation

Full technical docs: [docs.atas.net](https://docs.atas.net/)

Individual indicators reference help links: `[HelpLink("https://help.atas.net/support/...")]`
