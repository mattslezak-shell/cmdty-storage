# Cmdty Storage 
[![Build Status](https://dev.azure.com/cmdty/github/_apis/build/status/cmdty.storage?branchName=master)](https://dev.azure.com/cmdty/github/_build/latest?definitionId=2&branchName=master)
![Azure DevOps coverage](https://img.shields.io/azure-devops/coverage/cmdty/github/2)
[![NuGet](https://img.shields.io/nuget/v/cmdty.storage.svg)](https://www.nuget.org/packages/Cmdty.Storage/)
[![PyPI](https://img.shields.io/pypi/v/cmdty-storage.svg)](https://pypi.org/project/cmdty-storage/)

Multi-Factor Least Squares Monte Carlo energy storage valuation model.

### Table of Contents
* [Overview](#overview)
* [Models Implemented](#models-implemented)
* [Getting Started](#getting-started)
    * [Installing C# API](#installing-c-api)
    * [Installing Python Package](#installing-python-package)
* [Using the Python API](#using-the-python-api)
    * [Creating the Storage Object](#creating-the-storage-object)
        * [Storage with Constant Parameters](#storage-with-constant-parameters)
        * [Storage with Time and Inventory Varying Inject/Withdraw Rates](#storage-with-time-and-inventory-varying-injectwithdraw-rates)
    * [Storage Optimisation Using LSMC](#storage-optimisation-using-lsmc)
    * [Inspecting Valuation Results](#inspecting-valuation-results)
    * [Example Python GUI](#example-python-gui)
* [Using the C# API](#using-the-c-api)
    * [Creating the Storage Object](#creating-the-storage-object-1)
        * [Storage with Constant Parameters](#storage-with-constant-parameters-1)
        * [Storage with Time and Inventory Varying Inject/Withdraw Rates](#storage-with-time-and-inventory-varying-injectwithdraw-rates-1)
    * [Calculating Optimal Storage Value](#calculating-optimal-storage-value)
        * [Calculating the Intrinsic Value](#calculating-the-intrinsic-value)
        * [Calculating the Extrinsic Value: One-Factor Trinomial Tree](#calculating-the-extrinsic-value-one-factor-trinomial-tree)
* [Building](#building)
    * [Build on Windows](#building-on-windows)
        * [Build Prerequisites](#build-prerequisites)
        * [Running the Build](#running-the-build)
        * [Build Artifacts](#build-artifacts)
    * [Building on Linux or macOS](#building-on-linux-or-macOS)
        * [Build Prerequisites](#build-prerequisites-1)
        * [Running the Build](#running-the-build-1)
        * [Build Artifacts](#build-artifacts-1)
* [Why the Strange Tech Stack?](#why-the-strange-tech-stack)
* [License](#license)

## Overview
A collection of models for the valuation and optimisation of commodity storage, either virtual or physical. The models can be used for any commodity, although are most suitable for natural gas storage valuation and optimisation.

Calculations take into account many of the complex features of physical storage including:
* Inventory dependent injection and withdrawal rates, otherwise known as ratchets. For physical storage it is often the case that maximum withdrawal rates will increase, and injection rates will decrease as the storage inventory increases. For natural gas, this due to the increased pressure within the storage cavern.
* Time dependent injection and withdrawal rates, including the ability to add outages when no injection or withdrawal is allowed.
* Forced injection/withdrawal, as can be enforced by regulatory or physical constraints.
* Commodity consumed on injection/withdrawal, for example where natural gas is consumed by the motors that power injection into storage.
* Time dependent minimum and maximum inventory, necessary if different notional volumes of a storage facility are leased for different consecutive years.
* Optional time and inventory dependent loss of commodity in storage. For example this assumption is necessary for electricity storage which isn't 100% efficient.
* Ability to constrain the storage to be empty at the end of it's life, or specify a value of commodity inventory left in storage.

## Models Implemented
Currently the following models are implemented in this repository:
* Intrinsic valuation, i.e. optimal value assuming the commodity price remains static.
* One-factor trinomial tree, with seasonal spot volatility.
* Least-Squares Monte Carlo with a multi-factor price process.

## Getting Started

### Installing C# API
For use from C# install the NuGet package Cmdty.Storage.
```
PM> Install-Package Cmdty.Storage -Version 0.1.0-beta2
```

### Installing Python Package

```
> pip install cmdty-storage
```

## Using the Python API

### Creating the Storage Object
The first step is to create an instance of the class CmdtyStorage which
represents the storage facility. This section gives two examples of how to do this.
For full details on how to create CmdtyStorage instances see the Jupyter notebook 
[creating_storage_instances.ipynb](./samples/python/creating_storage_instances.ipynb).

#### Storage with Constant Parameters
The following code creates a simple storage object with constant constraints.

```python
from cmdty_storage import CmdtyStorage, RatchetInterp
import pandas as pd
storage_simple = CmdtyStorage(
    freq='D',
    storage_start = '2021-04-01',
    storage_end = '2022-04-01',
    injection_cost = 0.01,
    withdrawal_cost = 0.025,
    min_inventory = 0.0,
    max_inventory = 1500.0,
    max_injection_rate = 25.5,
    max_withdrawal_rate = 30.9
)
```

#### Storage with Time and Inventory Varying Inject/Withdraw Rates
The second examples creates a storage object with inventory-varying injection and
withdrawal rates, commonly known as "ratchets".

```python
storage_with_ratchets = CmdtyStorage(
    freq='D',
    storage_start = '2021-04-01',
    storage_end = '2022-04-01',
    injection_cost = 0.01,
    withdrawal_cost = 0.025,
    ratchets = [
                ('2021-04-01', # For days after 2021-04-01 (inclusive) until 2022-10-01 (exclusive):
                       [
                            (0.0, -150.0, 250.0),    # At min inventory of zero, max withdrawal of 150, max injection 250
                            (2000.0, -200.0, 175.0), # At inventory of 2000, max withdrawal of 200, max injection 175
                            (5000.0, -260.0, 155.0), # At inventory of 5000, max withdrawal of 260, max injection 155
                            (7000.0, -275.0, 132.0), # At max inventory of 7000, max withdrawal of 275, max injection 132
                        ]),
                  ('2022-10-01', # For days after 2022-10-01 (inclusive):
                       [
                            (0.0, -130.0, 260.0),    # At min inventory of zero, max withdrawal of 130, max injection 260
                            (2000.0, -190.0, 190.0), # At inventory of 2000, max withdrawal of 190, max injection 190
                            (5000.0, -230.0, 165.0), # At inventory of 5000, max withdrawal of 230, max injection 165
                            (7000.0, -245.0, 148.0), # At max inventory of 7000, max withdrawal of 245, max injection 148
                        ]),
                 ],
    ratchet_interp = RatchetInterp.LINEAR
)
```


### Storage Optimisation Using LSMC
The following is an example of valuing the storage using LSMC and a [three-factor seasonal model](https://github.com/cmdty/core/blob/master/docs/three_factor_seasonal_model/three_factor_seasonal_model.pdf) of price dynamics.
For comprehensive documentation of invoking the LSMC model, using both the three-factor price 
model, and a more general multi-factor model, see the notebook [multifactor_storage.ipynb](./samples/python/multifactor_storage.ipynb).

```python
from cmdty_storage import three_factor_seasonal_value

# Creating the Inputs
monthly_index = pd.period_range(start='2021-04-25', periods=25, freq='M')
monthly_fwd_prices = [16.61, 15.68, 15.42, 15.31, 15.27, 15.13, 15.96, 17.22, 17.32, 17.66, 
                      17.59, 16.81, 15.36, 14.49, 14.28, 14.25, 14.32, 14.33, 15.30, 16.58, 
                      16.64, 16.79, 16.64, 15.90, 14.63]
fwd_curve = pd.Series(data=monthly_fwd_prices, index=monthly_index).resample('D').fillna('pad')

rates = [0.005, 0.006, 0.0072, 0.0087, 0.0101, 0.0115, 0.0126]
rates_pillars = pd.PeriodIndex(freq='D', data=['2021-04-25', '2021-06-01', '2021-08-01', '2021-12-01', '2022-04-01', 
                                              '2022-12-01', '2023-12-01'])
ir_curve = pd.Series(data=rates, index=rates_pillars).resample('D').asfreq('D').interpolate(method='linear')

def settlement_rule(delivery_date):
    return delivery_date.asfreq('M').asfreq('D', 'end') + 20

# Call the three-factor seasonal model
three_factor_results = three_factor_seasonal_value(
    cmdty_storage = storage_with_ratchets,
    val_date = '2021-04-25',
    inventory = 1500.0,
    fwd_curve = fwd_curve,
    interest_rates = ir_curve,
    settlement_rule = settlement_rule,
    num_sims = 2000,
    seed = 12,
    spot_mean_reversion = 91.0,
    spot_vol = 0.85,
    long_term_vol =  0.30,
    seasonal_vol = 0.19,
    basis_funcs = '1 + x_st + x_sw + x_lt + s + x_st**2 + x_sw**2 + x_lt**2 + s**2 + s * x_st',
    discount_deltas = True
)

# Inspect the NPV results
print('Full NPV:\t{0:,.0f}'.format(three_factor_results.npv))
print('Intrinsic NPV: \t{0:,.0f}'.format(three_factor_results.intrinsic_npv))
print('Extrinsic NPV: \t{0:,.0f}'.format(three_factor_results.extrinsic_npv))
```
Prints the following.
```
Full NPV:	78,175
Intrinsic NPV: 	40,976
Extrinsic NPV: 	37,199
```

### Inspecting Valuation Results
The object returned from the calling `three_factor_seasonal_value` has many properties containing useful information. The code below give examples of a
few of these. See the **Valuation Results** section of [multifactor_storage.ipynb](./samples/python/multifactor_storage.ipynb) for more details.

Plotting the daily Deltas and projected inventory:
```python
%matplotlib inline
ax_deltas = three_factor_results.deltas.plot(title='Daily Deltas vs Projected Inventory', legend=True, label='Delta')
ax_deltas.set_ylabel('Delta')
inventory_projection = three_factor_results.expected_profile['inventory']
ax_inventory = inventory_projection.plot(secondary_y=True, legend=True, ax=ax_deltas, label='Expected Inventory')
h1, l1 = ax_deltas.get_legend_handles_labels()
h2, l2 = ax_inventory.get_legend_handles_labels()
ax_inventory.set_ylabel('Inventory')
ax_deltas.legend(h1+h2, l1+l2, loc=1)
```

![Delta Chart](./assets/delta_inventory_chart.png)

The **trigger_prices** property contains information on "trigger prices" which are approximate spot price levels at which the exercise decision changes.
* The withdraw trigger price is the spot price level, at time of nomination, above which the optimal decision will change to withdraw.
* The inject trigger price is the spot price level, at time of nomination, below which the optimal decision will change to inject.

Plotting the trigger prices versus the forward curve:
```python
%matplotlib inline
ax_triggers = three_factor_results.trigger_prices['inject_trigger_price'].plot(
    title='Trigger Prices vs Forward Curve', legend=True)
three_factor_results.trigger_prices['withdraw_trigger_price'].plot(legend=True)
fwd_curve['2021-04-25' : '2022-04-01'].plot(legend=True)
ax_triggers.legend(['Inject Trigger Price', 'Withdraw Trigger', 'Forward Curve'])
```
![Trigger Prices Chart](./assets/trigger_prices_chart.png)

### Example Python GUI
An example GUI notebook created using Jupyter Widgets can be found 
[here](./samples/python/multi_factor_gui.ipynb).

![Demo GUI](./assets/gui_demo.gif)


## Using the C# API

### Creating the Storage Object
In order for storage capacity to be valued, first an instance of the class CmdtyStorage 
needs to be created. The code samples below shows how the fluent builder API can be used
to achieve this. Once the Cmdty.Storage package has been installed,
a good way to discover the flexibility in the API is to look at the IntelliSense suggestions in
Visual Studio.

#### Storage with Constant Parameters
The code below shows simple storage facility with constant parameters.

``` c#
const double constantMaxInjectRate = 5.26;
const double constantMaxWithdrawRate = 14.74;
const double constantMaxInventory = 1100.74;
const double constantMinInventory = 0.0;
const double constantInjectionCost = 0.48;
const double constantWithdrawalCost = 0.74;

CmdtyStorage<Day> storage = CmdtyStorage<Day>.Builder
    .WithActiveTimePeriod(new Day(2019, 9, 1), new Day(2019, 10, 1))
    .WithConstantInjectWithdrawRange(-constantMaxWithdrawRate, constantMaxInjectRate)
    .WithConstantMinInventory(constantMinInventory)
    .WithConstantMaxInventory(constantMaxInventory)
    .WithPerUnitInjectionCost(constantInjectionCost, injectionDate => injectionDate)
    .WithNoCmdtyConsumedOnInject()
    .WithPerUnitWithdrawalCost(constantWithdrawalCost, withdrawalDate => withdrawalDate)
    .WithNoCmdtyConsumedOnWithdraw()
    .WithNoCmdtyInventoryLoss()
    .WithNoCmdtyInventoryCost()
    .MustBeEmptyAtEnd()
    .Build();
```

#### Storage with Time and Inventory Varying Inject/Withdraw Rates
The code below shows how to create a more complicated storage object with injection/withdrawal 
rates being dependent on time and the inventory level.This is much more respresentative of real 
physical gas storage capacity.

``` c#
const double constantInjectionCost = 0.48;
const double constantWithdrawalCost = 0.74;

var injectWithdrawConstraints = new List<InjectWithdrawRangeByInventoryAndPeriod<Day>>
{
    (period: new Day(2019, 9, 1), injectWithdrawRanges: new List<InjectWithdrawRangeByInventory>
    {
        (inventory: 0.0, (minInjectWithdrawRate: -44.85, maxInjectWithdrawRate: 56.8)), // Inventory empty, highest injection rate
        (inventory: 100.0, (minInjectWithdrawRate: -45.01, maxInjectWithdrawRate: 54.5)),
        (inventory: 300.0, (minInjectWithdrawRate: -45.78, maxInjectWithdrawRate: 52.01)),
        (inventory: 600.0, (minInjectWithdrawRate: -46.17, maxInjectWithdrawRate: 51.9)),
        (inventory: 800.0, (minInjectWithdrawRate: -46.99, maxInjectWithdrawRate: 50.8)),
        (inventory: 1000.0, (minInjectWithdrawRate: -47.12, maxInjectWithdrawRate: 50.01)) // Inventory full, highest withdrawal rate
    }),
    (period: new Day(2019, 9, 20), injectWithdrawRanges: new List<InjectWithdrawRangeByInventory>
    {
        (inventory: 0.0, (minInjectWithdrawRate: -31.41, maxInjectWithdrawRate: 48.33)), // Inventory empty, highest injection rate
        (inventory: 100.0, (minInjectWithdrawRate: -31.85, maxInjectWithdrawRate: 43.05)),
        (inventory: 300.0, (minInjectWithdrawRate: -31.68, maxInjectWithdrawRate: 41.22)),
        (inventory: 600.0, (minInjectWithdrawRate: -32.78, maxInjectWithdrawRate: 40.08)),
        (inventory: 800.0, (minInjectWithdrawRate: -33.05, maxInjectWithdrawRate: 39.74)),
        (inventory: 1000.0, (minInjectWithdrawRate: -34.80, maxInjectWithdrawRate: 38.51)) // Inventory full, highest withdrawal rate
    })
};

CmdtyStorage<Day> storage = CmdtyStorage<Day>.Builder
    .WithActiveTimePeriod(new Day(2019, 9, 1), new Day(2019, 10, 1))
    .WithTimeAndInventoryVaryingInjectWithdrawRates(injectWithdrawConstraints)
    .WithPerUnitInjectionCost(constantInjectionCost, injectionDate => injectionDate)
    .WithNoCmdtyConsumedOnInject()
    .WithPerUnitWithdrawalCost(constantWithdrawalCost, withdrawalDate => withdrawalDate)
    .WithNoCmdtyConsumedOnWithdraw()
    .WithNoCmdtyInventoryLoss()
    .WithNoCmdtyInventoryCost()
    .MustBeEmptyAtEnd()
    .Build();
```

### Calculating Optimal Storage Value

#### Calculating the Intrinsic Value
The following example shows how to calculate the intrinsic value of the storage, including
the optimal intrinsic inject/withdraw decision profile.

``` c#
var currentPeriod = new Day(2019, 9, 15);
const double lowerForwardPrice = 56.6;
const double forwardSpread = 87.81;
double higherForwardPrice = lowerForwardPrice + forwardSpread;

var forwardCurveBuilder = new TimeSeries<Day, double>.Builder();

foreach (var day in new Day(2019, 9, 15).EnumerateTo(new Day(2019, 9, 22)))
{
    forwardCurveBuilder.Add(day, lowerForwardPrice);
}

foreach (var day in new Day(2019, 9, 23).EnumerateTo(new Day(2019, 10, 1)))
{
    forwardCurveBuilder.Add(day, higherForwardPrice);
}

const double startingInventory = 50.0;

IntrinsicStorageValuationResults<Day> valuationResults = IntrinsicStorageValuation<Day>
    .ForStorage(storage)
    .WithStartingInventory(startingInventory)
    .ForCurrentPeriod(currentPeriod)
    .WithForwardCurve(forwardCurveBuilder.Build())
    .WithCmdtySettlementRule(day => day.First<Month>().Offset(1).First<Day>().Offset(5))
    .WithDiscountFactorFunc(day => 1.0)
    .WithFixedGridSpacing(10.0)
    .WithLinearInventorySpaceInterpolation()
    .WithNumericalTolerance(1E-12)
    .Calculate();

Console.WriteLine("Calculated intrinsic storage NPV: " + valuationResults.NetPresentValue.ToString("F2"));
Console.WriteLine();
Console.WriteLine("Decision profile:");
Console.WriteLine(valuationResults.DecisionProfile.FormatData("F2", -1));
```

When run, the above code prints the following to the console.

```
Calculated intrinsic storage NPV: 10827.21

Decision profile:
Count = 16
2019-09-15  5.26
2019-09-16  5.26
2019-09-17  5.26
2019-09-18  5.26
2019-09-19  5.26
2019-09-20  5.26
2019-09-21  5.26
2019-09-22  5.26
2019-09-23  -14.74
2019-09-24  -14.74
2019-09-25  0.00
2019-09-26  -14.74
2019-09-27  -14.74
2019-09-28  -14.74
2019-09-29  -14.74
2019-09-30  -3.64
```

#### Calculating the Extrinsic Value: One-Factor Trinomial Tree
The code sample below shows how to calculate the optimal storage value, including extrinsic
option value, using a one-factor trinomial tree model.

``` c#
var currentPeriod = new Day(2019, 9, 15);

const double lowerForwardPrice = 56.6;
const double forwardSpread = 87.81;

double higherForwardPrice = lowerForwardPrice + forwardSpread;

var forwardCurveBuilder = new TimeSeries<Day, double>.Builder();

foreach (var day in new Day(2019, 9, 15).EnumerateTo(new Day(2019, 9, 22)))
{
    forwardCurveBuilder.Add(day, lowerForwardPrice);
}

foreach (var day in new Day(2019, 9, 23).EnumerateTo(new Day(2019, 10, 1)))
{
    forwardCurveBuilder.Add(day, higherForwardPrice);
}

TimeSeries<Month, Day> cmdtySettlementDates = new TimeSeries<Month, Day>.Builder
    {
        {new Month(2019, 9), new Day(2019, 10, 20) }
    }.Build();

const double interestRate = 0.025;

// Trinomial tree model parameters
const double spotPriceMeanReversion = 5.5;
const double onePeriodTimeStep = 1.0 / 365.0;

TimeSeries<Day, double> spotVolatility = new TimeSeries<Day, double>.Builder
    {
        {new Day(2019, 9, 15),  0.975},
        {new Day(2019, 9, 16),  0.97},
        {new Day(2019, 9, 17),  0.96},
        {new Day(2019, 9, 18),  0.91},
        {new Day(2019, 9, 19),  0.89},
        {new Day(2019, 9, 20),  0.895},
        {new Day(2019, 9, 21),  0.891},
        {new Day(2019, 9, 22),  0.89},
        {new Day(2019, 9, 23),  0.875},
        {new Day(2019, 9, 24),  0.872},
        {new Day(2019, 9, 25),  0.871},
        {new Day(2019, 9, 26),  0.870},
        {new Day(2019, 9, 27),  0.869},
        {new Day(2019, 9, 28),  0.868},
        {new Day(2019, 9, 29),  0.867},
        {new Day(2019, 9, 30),  0.866},
        {new Day(2019, 10, 1),  0.8655}
    }.Build();

const double startingInventory = 50.0;

TreeStorageValuationResults<Day> valuationResults = TreeStorageValuation<Day>
    .ForStorage(storage)
    .WithStartingInventory(startingInventory)
    .ForCurrentPeriod(currentPeriod)
    .WithForwardCurve(forwardCurveBuilder.Build())
    .WithOneFactorTrinomialTree(spotVolatility, spotPriceMeanReversion, onePeriodTimeStep)
    .WithMonthlySettlement(cmdtySettlementDates)
    .WithAct365ContinuouslyCompoundedInterestRate(settleDate => interestRate)
    .WithFixedGridSpacing(10.0)
    .WithLinearInventorySpaceInterpolation()
    .WithNumericalTolerance(1E-12)
    .Calculate();

Console.WriteLine("Calculated storage NPV: " + valuationResults.NetPresentValue.ToString("F2"));
```

The above code prints the following.

```
Calculated storage NPV: 24809.48
```

## Building
This section describes how to run a scripted build on a cloned repo. Visual Studio 2019 is used for development, and can also be used to build the C# and run unit tests on the C# and Python APIs. However, the scripted build process also creates packages (NuGet and Python), builds the C# samples, and verifies the C# interactive documentation. [Cake](https://github.com/cake-build/cake) is used for running scripted builds. The ability to run a full scripted build on non-Windows is [planned](https://github.com/cmdty/storage/issues/2), but at the moment it can only be done on Windows.

### Building on Windows

#### Build Prerequisites
The following are required on the host machine in order for the build to run.
* The .NET Core SDK. Check the [global.json file](global.json) for the version necessary, taking into account [the matching rules used](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json#matching-rules).
* The Python interpretter, accessible by being in a file location in the PATH environment variable. Version 3.6 is used, although other 3.x versions might work.
* The following Python packages installed:
    * virtualenv.
    * setuptools.
    * wheel.

#### Running the Build
The build is started by running the PowerShell script build.ps1 from a PowerShell console, ISE, or the Visual Studio Package Manager Console.

```
PM> .\build.ps1
```

#### Build Artifacts
The following results of the build will be saved into the artifacts directory (which itelf will be created in the top directory of the repo).
* The NuGet package: Cmdty.Storage.[version].nupkg
* The Python package files:
    * cmdty_storage-[version]-py3-none-any.whl
    * cmdty_storage-[version].tar.gz
* 32-bit and 64-bit versions of the Excel add-in:
    * Cmdty.Storage-x86.xll
    * Cmdty.Storage-x64.xll

### Building on Linux or macOS
At the moment only building, testing and packaging the .NET components is possible on a non-Windows OS.

#### Build Prerequisites
The following are required on the host machine in order for the build to run.
* The .NET Core SDK. Check the [global.json file](global.json) for the version necessary, taking into account [the matching rules used](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json#matching-rules).

#### Running the Build
Run the following commands in a cloned repo
```
> dotnet build src/Cmdty.Storage/ -c Release
> dotnet test tests/Cmdty.Storage.Test/ -c Release
> dotnet pack src/Cmdty.Storage -o artifacts -c Release --no-build
```

#### Build Artifacts
The following results of the build will be saved into the artifacts directory (which itelf will be created in the top directory of the repo).
* The NuGet package: Cmdty.Storage.[version].nupkg

## Why the Strange Tech Stack?
Users of the Python API might be perplexed as to the technology used: Python calling into .NET, which itself calls into native code for the Intel MKL numerical routines.
This is certainly not a common structure, especially for a package focussed on complex numerical calculations.
Where the Python runtime speed is an issue, as is suspected with this project, it is more usual to have a structure
where Python calls into native code using ctypes, or makes use of a [Numba](https://numba.pydata.org/).

However, the Cmdty project started off as a .NET only project, written in C#, due to the author being mainly
a C# guy during the day-job. The Python wrapper was added later as it became apparent that there was a demand to
use the models from Python. Since then it now seems that there are many more users of the Python API than
the C# NuGet package, resulting in significant time being spent on the Python API, and examples.

If the project was started again from scratch, potentially it would have been written entirely in Python
utilising Numba. However, due to time constraints, and not wanting to have the maintenance headache of 
having two independent implementations side-by-side there is no plan to work on this. That said,
if others want to have a go at a pure Python implementation it would be very much welcomed and I would
happily help out.

Despite the oddity in the structure it seems to work quite well with the performance of the LSMC
model being decent. Although compiled C# usually does not run as quickly as native code,
it's performance isn't bad at all, and the majority of the running time is spent during the QR 
decomposition for the regression which is itself done using Intel MKL, which does these calculations
pretty much as quickly as you can get. The only real annoyances with the structure is:
* [pythonnet](https://github.com/pythonnet/pythonnet) not currently supporting .NET Core. A fix for
this is currently [in the pipeline](https://github.com/pythonnet/pythonnet/issues/984) for the next pythonnet release. At current this means that Mono needs to be installed in order to use the Python API on Linux.
* The PyPI package size.
* If a version of the [curves](https://pypi.org/project/curves/) package is installed which has
a .NET dependency with a different version to a dependency of the cmdty-storage package 
 this can cause strange errors.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
