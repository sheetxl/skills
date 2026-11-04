---
name: charts
description: "SheetXL charts: reading what a chart plots, inserting one, and editing it — type, data, title, legend, axes, series, points, data labels, trendlines, error bars — through the chart API, Office Scripts' names. Load when a task mentions a chart. Triggers on: chart, getCharts, IChart, series, axis, legend, data labels, trendline, error bars, plot, graph."
---

# Charts — Read, Insert, Edit

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing. Code below
> is run-context code (`scripting.md`): snippet form — no `await` needed on the
> statements shown as plain calls (every statement auto-awaits); `getCharts().add`
> returns a promise and is awaited like any other.

**In this file:** What a chart is here · Reading a chart · Inserting one ·
Editing: the chart, its series, axes, elements · What is refused · Verify

## What a chart is here

A chart is a movable (`sheet.getMovables()`, type `chart`) whose content is an
`IChart`. It plots cells: every series states its name, categories and values as
formulas (`Data!$B$2:$B$7`), the way Excel's own dialog does. The API follows
Office Scripts' names (`getSeries`, `setChartType`, `getAxes().getValueAxis()`,
`getDataLabels`, `addChartTrendline`), so what you know from `ExcelScript.Chart`
carries over; VBA's names are noted where they differ.

Every setter writes exactly what Excel writes for the same change and is one step
to undo. A chart edited here opens in Excel as if Excel had made the edit.

## Reading a chart

The reports already say what a chart plots: the sheet report's `Charts` section
lists each chart with `series`, `names`, `categories`; a chart's own report
(`renderMovable`) adds its title, legend, axes and the analysis on its series
(`reading.md`). For more, ask the chart:

```typescript
const chart = sheet.getCharts().getItems()[0];   // or sheet.getCharts().getByName('Chart 2')
chart.getChartType();                             // 'columnClustered', 'line', 'pie', 'combo', …
chart.getChartTitle();                            // the text, a cell's text when it reads one
chart.getSeries().map(s => [s.getName(), s.getValuesFormula(), s.getCategoriesFormula()]);
chart.getAxes().getItems().map(a => [a.getKind(), a.getTitle().getText(), a.getNumberFormat()]);
chart.getLegend().getPosition();                  // 'bottom' | 'right' | 'top' | 'left' | 'corner'
```

A series' `getValuesRange()` is the cells, or `null` when the formula names
none (`#REF!`, a name, a literal `{1,2,3}`). A pivot chart (`getPivotSource()`)
states its table; its ranges are wherever the table last put its cells.

## Inserting a chart

```typescript
const chart = await sheet.getCharts().add('A1:D7', { type: 'line' });
```

The cells are read by Excel's rules: a header row names the series, the first
column holds the categories, and the series are the columns unless there are more
columns than rows (`seriesBy: 'rows' | 'columns'` to say). `type` is any type
Excel's gallery offers — `columnClustered`, `columnStacked`, `columnStacked100`,
`bar…`, `line`, `lineMarkers`, `area…`, `pie`, `doughnut`, `xyscatter…`,
`bubble`, `radar…`, `stockVHLC`…, the 3-D forms (`3DColumnClustered`, `3DPie`…),
the surfaces, and the 2016 types (`waterfall`, `treemap`, `sunburst`, `histogram`,
`pareto`, `boxwhisker`, `funnel`, `regionMap`), which are made and saved as Excel
makes them but drawn as a placeholder for now. The chart is 5 by 3 inches in the
middle of the window unless `bounds` says otherwise.

## Editing

### The chart

```typescript
chart.setChartType('lineMarkers');          // every series; a combo becomes one group
chart.setData('Data!A1:E7', 'columns');     // other cells, same formatting where series take their place
chart.setSeriesBy('rows');                  // Switch Row/Column
chart.update({ title: 'Sales by Month', legend: { position: 'right' } });
chart.getTitle().setText('=Data!$A$1');     // a title that follows a cell
chart.setElement('legendTop');              // any Add Chart Element pick, by VBA's msoElement… name
chart.update({ style: { fill: '#F2F2F2', font: { family: 'Arial', size: 11 } }, roundedCorners: true });
chart.getDataTable().update({ visible: true, showLegendKey: true });
chart.getDataLabels().setVisible(true);     // every series' labels
chart.getMovable().setBounds({ x: 480, y: 20, width: 480, height: 288 }); // px; placement is the movable's
```

`update` takes several properties at once as one undo step. `setElement` takes
the element names of `MsoChartElementType` without the prefix, in lower camel
case: `chartTitleAboveChart`, `dataLabelOutsideEnd`, `primaryValueGridLinesMinor`,
`primaryCategoryAxisTitleBelowAxis`, `trendlineAddLinear`, `errorBarStandardError`,
`upDownBarsShow`, `dataTableWithLegendKeys`, …

### Series and points

```typescript
const s = chart.getSeries()[0];
s.update({ name: 'Total', values: 'D2:D7' });
s.setFormula('=SERIES(Data!$D$1,Data!$A$2:$A$7,Data!$D$2:$D$7,1)');
s.setChartType('line'); s.setAxisGroup('secondary');     // a combo
s.update({ style: { fill: '#FF0000', stroke: { width: 2 } }, gapWidth: 50, smooth: true });
s.getDataLabels().update({ visible: true, position: 'outsideEnd', numberFormat: '0.0', showPercentage: true });
s.getPoints()[2].update({ style: { fill: '#00B050' }, explosion: 20, hasDataLabel: true });
s.addChartTrendline('polynomial').update({ polynomialOrder: 3, showEquation: true, showRSquared: true });
s.getYErrorBars().update({ type: 'stDev', amount: 1 });
s.getMarker().update({ symbol: 'diamond', size: 8 });
chart.addSeries('East').update({ values: 'D2:D7' });
s.delete();
```

A series' categories are shared by every series on its axis: setting them on one
sets them all (as Excel does). `setFormula` takes Excel's `=SERIES(name, categories,
values, order[, sizes])`, every part optional.

### Axes

```typescript
const value = chart.getAxes().getValueAxis();          // the primary; getChartAxis('value', 'secondary') for the other
value.update({ minimum: 0, maximum: 40, majorUnit: 10, minorUnit: 2, numberFormat: '#,##0', title: 'Units' });
value.update({ scaleType: 'logarithmic', logBase: 10, displayUnit: 'thousands', showDisplayUnitLabel: true });
value.update({ reversePlotOrder: true, crosses: 'maximum', majorTickMark: 'cross', tickLabelPosition: 'high' });
const category = chart.getAxes().getCategoryAxis();
category.update({ majorGridlines: true, textOrientation: 45, categoryNames: 'H2:I7', multiLevel: true });
category.update({ categoryType: 'dateAxis', baseTimeUnit: 'days', majorUnit: 1, majorTimeUnitScale: 'months' });
value.getTitle().update({ text: 'Units', textOrientation: 0, style: { font: { bold: true } } });
```

`minimum`, `maximum` and the units read as the layout draws them and take
`'auto'` to be automatic again.

### Elements

Every part of a chart is an element with a path, and the chart answers the path
with the same object for as long as the part is there:

```typescript
chart.getElements().map(e => e.getPath());   // what the chart shows: 'title', 'axes/primary/value', 'series/0', …
const t = chart.getElement('axes/primary/value/title');   // null where there is none
t?.getElementType();                          // 'axisTitle'
chart.getElement('series/0/points/2')?.getStyle?.()?.setFill('#00B050');
```

Paths: `chartArea`, `title`, `legend`, `legend/entries/<n>`, `plotArea`,
`dataTable`, `axes/<primary|secondary>/<category|value|series>` (with `/title`,
`/majorGridlines`, `/minorGridlines`), `series/<key>` (with `/marker`,
`/dataLabels`, `/errorBars/<x|y>`, `/trendlines/<n>`, `/points/<n>`,
`/points/<n>/marker`, `/points/<n>/dataLabel`), `groups/<n>` (with
`/upDownBars`, `/hiLoLines`, `/dropLines`, `/seriesLines`). A series' key is its
`c:idx`; points, trendlines and groups go by their place, from 0. Each element
has a type flag (`isISeries`, `isIAxis`, `isITitle`, …) and, where it is drawn,
`getStyle()`: fill and line (`IShapeStyle`), and on a part with text (chart area,
title, axis title, legend, legend entry, axis, data labels, data table) also
`getStyle().getFont()` (`ITextBoxStyle`, `isITextBoxStyle`). There is no
`getFont()` on a part. The font is an `IFontStyle` (family, size, bold, italic,
underline, strike, color, letter spacing, baseline), the same interface a cell's
`range.getStyle().getFont()` answers, so the same code formats either; `update`
nests it the same way: `title.update({ style: { fill: 'white', font: { bold: true } } })`.
`chart.getStyle().getFont()` sets every text in the chart. A legend entry's fill
and line are the legend's; setting them throws. A theme font is the family
`'theme:body'` or `'theme:heading'`.

## What is refused

Each refusal is an `ExpectedError` saying why; nothing is written.

- A 2016 type (waterfall, histogram, pareto, box and whisker, treemap, sunburst,
  funnel, region map): its title, legend and type are edited; the rest is not yet,
  and it is drawn as a placeholder.
- A pivot chart: it changes with its table.
- A property Excel refuses for the type: a trendline on a pie, drop lines on a
  column chart, a bubble size label on a line, `setChartType` to a stock type.
- An address that is not one. A reference to cells that are gone (`#REF!`) is kept,
  as Excel keeps it, and the series plots nothing.

## Verify

Read back what you set — `chart.getChartType()`, `series.getValuesFormula()`,
`axis.getMinimum()` — and the chart's report (`renderMovable`). Saved to `.xlsx`, a
chart reads back as the same chart and opens in Excel without repair.
