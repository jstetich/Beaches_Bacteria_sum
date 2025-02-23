Graphics for Bacteria Levels at Casco Bay Beaches
================
Curtis C. Bohlen, Casco Bay Estuary Partnership.
01/23/2021

-   [Introduction](#introduction)
    -   [Handling non-detects](#handling-non-detects)
    -   [Standards](#standards)
        -   [Beaches Program](#beaches-program)
        -   [Maine State Class SB Waters
            Standards](#maine-state-class-sb-waters-standards)
-   [Import Libraries](#import-libraries)
-   [Data Preparation](#data-preparation)
    -   [Initial Folder References](#initial-folder-references)
    -   [Load Data](#load-data)
    -   [Add a “Beach” Identifier](#add-a-beach-identifier)
    -   [Simplify and Correct Beach
        Names](#simplify-and-correct-beach-names)
    -   [Add a “Day of the Week”
        Identifier](#add-a-day-of-the-week-identifier)
    -   [Add Maximum Likelihood Estimate for
        Non-detects](#add-maximum-likelihood-estimate-for-non-detects)
    -   [Calculate Exceedences](#calculate-exceedences)
    -   [Restrict to Recent Conditions](#restrict-to-recent-conditions)
-   [Create Geometric Mean Function](#create-geometric-mean-function)
-   [Plots of Recent Condition](#plots-of-recent-condition)
    -   [Jitter Plot of Recent
        Conditions](#jitter-plot-of-recent-conditions)
        -   [Add Geometric Means](#add-geometric-means)
        -   [Add Annotations](#add-annotations)
    -   [Violin Plot with Jitter](#violin-plot-with-jitter)
        -   [Add Geometric Means](#add-geometric-means-1)
    -   [Boxplot with Jitter](#boxplot-with-jitter)
        -   [With Color by Rainfall](#with-color-by-rainfall)
        -   [With Color by Detection](#with-color-by-detection)
        -   [Add Annotation](#add-annotation)
        -   [Add Geometric Means](#add-geometric-means-2)
-   [Trend Graphics](#trend-graphics)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

We present code for generating draft graphics for the “State of Casco
Bay” report, relating specifically to levels of bacteria observed at
swimming beaches.

The Beaches program monitors bacteria levels (currently) at six Casco
Bay beaches. Data is collected periodically (usually weekly) at each
beach, to inform beach managers and the public about possible risk of
swimming in water that may be polluted by certain pathogens.

The Beaches program measures “enterococci” bacteria, while DMR’s
shellfish program monitor’s “fedal coliform” bacteria. the two measures
are generally correlated, but are not directly comparable because of
different methods.

## Handling non-detects

**All graphics here treat non-detects as equal to the Reporting Limit!**

We generally prefer to replace non-detects with an estimate of what
“would have been observed” if the methods were more sensitive. One can
generate such estimates (more correctly, expected values of such
estimates) based on certain statistical assumptions. While those
assumptions can be called into question, such methods are usually
preferable to ignoring non-detects, assuming they represent the value of
“true” measurements of zero, or making other *ad hoc* assumptions.

Unfortunately, we could not use statistical methods to correct for
non-detects with the data on bacteria levels at shellfish growing areas.
For technical reasons, some of the statistical methods we used to
analyze the shellfish data do not work with data that replaces
non-detects with maximum likelihood estimates of the (expected value of)
the unobserved values.

For consistency in presentation in the State of Casco Bay Report, all
GRAPHICS for both Beaches and Shellfish bacteria data are presented
using the same conventions, including using data that treats non-detects
as equal to their detection limits. Because bacteria data are so skewed
and heavy tailed, this has only modest effect on many summary
statistics. However it does alter values of low quantiles and geometric
means.

Most statistical testing for the Beaches data are based on data that
replaces non-detects with maximum likelihood estimators of their
conditional means, but thoseresults are never directly presented in
SoCB.

## Standards

### Beaches Program

104 CFU / 100 ml, for individual observations.

### Maine State Class SB Waters Standards

> the number of enterococcus bacteria in these waters may not exceed a
> geometric mean of 8 CFU per 100 milliliters in any 90-day interval or
> 54 CFU per 100 milliliters in more than 10% of the samples in any
> 90-day interval.

38 M.R.S. §465-B(2)(B)

# Import Libraries

``` r
library(fitdistrplus)
#> Loading required package: MASS
#> Loading required package: survival
library(tidyverse)
#> -- Attaching packages --------------------------------------- tidyverse 1.3.1 --
#> v ggplot2 3.3.5     v purrr   0.3.4
#> v tibble  3.1.6     v dplyr   1.0.7
#> v tidyr   1.1.4     v stringr 1.4.0
#> v readr   2.1.1     v forcats 0.5.1
#> -- Conflicts ------------------------------------------ tidyverse_conflicts() --
#> x dplyr::filter() masks stats::filter()
#> x dplyr::lag()    masks stats::lag()
#> x dplyr::select() masks MASS::select()

library(mgcv)      # For GAMs and GAMMs; used here for seasonal smoothers
#> Loading required package: nlme
#> 
#> Attaching package: 'nlme'
#> The following object is masked from 'package:dplyr':
#> 
#>     collapse
#> This is mgcv 1.8-38. For overview type 'help("mgcv-package")'.
library(emmeans)   # For marginal means

library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())

library(LCensMeans)
```

# Data Preparation

## Initial Folder References

``` r
sibfldnm    <- 'Data'
parent      <- dirname(getwd())
sibling     <- file.path(parent,sibfldnm)

dir.create(file.path(getwd(), 'figures'), showWarnings = FALSE)
#dir.create(file.path(getwd(), 'models'),  showWarnings = FALSE)
```

## Load Data

``` r
fn <- "beaches_data.csv"
beach_data <- read_csv(file.path(sibling, fn))
#> Rows: 1895 Columns: 24
#> -- Column specification --------------------------------------------------------
#> Delimiter: ","
#> chr   (8): SiteCode, Sample_ID, Sample_Qualifier, Lab_Qualifier, Weather, Pa...
#> dbl  (11): Year, Month, DOY, Enterococci, Reporting_Limit, Bacteria, Rain24,...
#> lgl   (3): Censored_Flag, Tide_Stage, Current
#> dttm  (1): sdatetime
#> date  (1): sdate
#> 
#> i Use `spec()` to retrieve the full column specification for this data.
#> i Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

## Add a “Beach” Identifier

``` r
fn = "beach_locations.csv"
beach_lookup = read_csv(file.path(sibling, fn),
                        col_types = cols(
                          Town = col_character(),
                          Beach_Name = col_character(),
                          SamplePoint = col_character(),
                          Latitude = col_double(),
                          Longitude = col_double()
                        )) %>%
  select(-Latitude, -Longitude)

beach_data <- beach_data %>%
  mutate(Beach = beach_lookup$Beach_Name[match(SiteCode, 
                                               beach_lookup$SamplePoint)])
```

## Simplify and Correct Beach Names

``` r
beach_data <- beach_data %>%
  mutate(Beach = if_else(Beach == 'Stovers Point Preserve', "Stover's Point", Beach)) %>%
  mutate(Beach = if_else(Beach == 'Broad Cove Reserve', "Broad Cove", Beach)) %>%
  mutate(Beach = if_else(Beach == 'Mitchell Field Beach', "Mitchell Field", Beach))
```

## Add a “Day of the Week” Identifier

We need this to help evaluate whether samples are “normal” samples or
“storm” samples.

``` r
beach_data <- beach_data %>%
  mutate(Weekday = weekdays(sdate)) %>%
  relocate(Weekday, .after = Month)
```

## Add Maximum Likelihood Estimate for Non-detects

This uses our LCensMeans package, and estimates a maximum likelihood
estimate of the expected value of the (unobserved) left censored values.
It relies on several assumption that are questionable for these data,
but it is arguably better than using the detection limit or half the
detection limit.

Unfortunately, some of the more sophisticated models we run on the
shellfish data do not work with data transformed in this way. For
consistency, all GRAPHICS for both BEaches and Shellfish bacteria data
are presented showing data treating non-detects at the nominal detection
limit. For the BEaches data, most of the analyses were based on data
that substituted ML estimators for the non-detects.

``` r
beach_data <- beach_data %>%
  mutate(Bacteria2 = sub_cmeans(Bacteria, Censored_Flag) )
```

## Calculate Exceedences

``` r
beach_data <- beach_data %>%
  mutate(Exceeds = Bacteria > 104) %>%
  relocate(Exceeds, .after = Censored_Flag)
```

## Restrict to Recent Conditions

``` r
recent_data <- beach_data %>%
  filter(Year > 2015)
```

# Create Geometric Mean Function

``` r
gm_mean <- function(x) {
  exp(mean(log(x)))
}
```

# Plots of Recent Condition

## Jitter Plot of Recent Conditions

``` r
jitter_plt <- recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria)) +
  
  geom_jitter(aes(color = Censored_Flag),
              width = 0.3, 
              height = .05,
              alpha = 0.5) +
  
  scale_y_log10() +
  scale_color_manual(values = cbep_colors(), 
                     name = '', labels = c('Observed', 'Below Detection')) +
  
  theme_cbep(base_size = 12) +

  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +
  theme(legend.position = c(.6, .9)) +
  
  guides(color = guide_legend(override.aes = list(alpha = c(0.5,0.751) ) )) +

  ylab('Enterococci (MPN / 100ml)') +
  xlab('')
```

### Add Geometric Means

This looks like the geometric means annotation are not properly lined
up, but it is lined up better in the PDF version, which is what counts.

``` r
xanchor <- 3.75
yanchor <- 2200

jitter_plt <- jitter_plt + 
  stat_summary(fun = gm_mean, fill = 'red',shape = 22) 
  
  # annotate('point', x= xanchor, y = yanchor,
  #           size = 3, pch = 22, fill = 'red') +
  # annotate('text', x= xanchor + 0.25, y = yanchor,
  #          hjust = 0, size = 3.5, label = 'Geometric Mean')
```

### Add Annotations

``` r
jitter_plt +
  geom_hline(yintercept = 104, color = 'gray25', lty = 2) +
  #geom_hline(yintercept = 8, color = 'gray25', lty = 2) +
  
  annotate('text', x = 0, y  = 130, label = '104 MPN', 
           size = 2.5, hjust = 0) +
  annotate('text', x = 3.33, y = 2355, label = 'Geometric Mean', 
           size = 3.5, hjust = 0) +
  annotate('point', x = 3.01, y = 2355, shape = 22, fill = 'red')
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/jitter_all-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/recent_conditons_jitter_revised.pdf', device = cairo_pdf, 
       width = 5, height = 4)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

## Violin Plot with Jitter

The challenge here is that if we make width proportional to number of
observations, the large number of observations and non-detect at
Stover’s Point dominates. So we make all sites equal width , and show
dots to signal sample size.

``` r
violin_plt <- recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria)) +
  geom_violin(scale = 'width',
              width = .75,
              # draw_quantiles = c(0.25, 0.5, 0.75),
              fill = cbep_colors()[6],
              color = 'gray75',
              alpha = 0.1) +
  geom_jitter(aes(color = Censored_Flag),
              width = 0.15, 
              height = .05,
              alpha = 0.5) +
  
  geom_hline(yintercept = 104, color = 'gray25', lty = 2) +
  annotate('text', x = 0, y  = 130, label = '104 MPN', size = 2.5, hjust = 0) +
  
  scale_y_log10() +
  scale_color_manual(values = cbep_colors(), 
                     name = '', labels = c('Observed', 'Below Detection')) +
  
  theme_cbep(base_size = 12) +

  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +
  theme(legend.position = c(.65, .9)) +

  guides(color = guide_legend(override.aes = list(alpha = c(0.5,0.75) ) )) +

  ylab('Enterococci (MPN / 100ml)') +
  xlab('')
```

### Add Geometric Means

``` r
xanchor <- 3.5
yanchor <- 1500

violin_plt + 
  stat_summary(fun = gm_mean, fill = 'red',shape = 22) 
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/violin_final-1.png" style="display: block; margin: auto;" />

``` r
  
  # annotate('point', x= xanchor, y = yanchor,
  #           size = 3, pch = 22, fill = 'red') +
  # annotate('text', x= xanchor + 0.3, y = yanchor,
  #          hjust = 0, size = 3.5, label = 'Geometric Mean')

ggsave('figures/recent_conditons_jitter_violin.pdf', device = cairo_pdf, 
       width = 5, height = 4)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

## Boxplot with Jitter

### With Color by Rainfall

``` r
recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria)) +
  geom_boxplot(width = .6, outlier.shape = NA) +
  geom_jitter(aes(color = Rain48), width = 0.25, 
              height = .05, 
              alpha = 1) +
  
  geom_hline(yintercept = 104, color = 'gray25', lty = 2) +
  #geom_hline(yintercept = 8, color = 'gray25', lty = 2) +
  
  scale_y_log10() +
  
  scale_color_viridis_c('Inches of Rain\n Prior 48 Hours') +
  
  theme_cbep(base_size = 12) +

  theme(legend.key.size = unit(10, 'points'),
        legend.title = element_text(size = 9)) +
  
  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +

  ylab('Enterococci (MPN / 100ml)') +
  xlab('') +
  stat_summary(fun = gm_mean, fill = 'red',shape = 22)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/boxplot_rainfall-1.png" style="display: block; margin: auto;" />

### With Color by Detection

``` r
plt <- recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria)) +
  geom_boxplot(width = .6, outlier.shape = NA) +
  geom_jitter(aes(color = Censored_Flag),
              width = 0.25, 
              height = .05, 
              #color = cbep_colors()[5],
              alpha = 0.5) +
  
  geom_hline(yintercept = 104, color = 'gray25', lty = 2) +
  annotate('text', x = 0, y  = 130, label = '104 MPN', size = 2.5, hjust = 0) +
  
  scale_y_log10() +
  
  scale_color_manual(values = cbep_colors(), 
                     labels = c('Observed', 'Below Detection'),
                     name = NULL) +
  
  theme_cbep(base_size = 12) +

  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +
  theme(legend.position = c(.65, .875),
        legend.text = element_text(size = 8),
        legend.key.height = unit(10, 'points')) +

  ylab('Enterococci (MPN / 100ml)') +
  xlab('')
```

### Add Annotation

``` r
# Calculate positions for annotations
xanchor <- 0.75

# On a log scale, equally spaced items have the same RATIO
yanchor <- 6000
ylow    <-  2/3*yanchor
yhigh    <-  3/2* yanchor

plt2 <- plt + 
  annotate('rect', xmin = xanchor, ymin = ylow, xmax =xanchor + 0.1, ymax = yhigh,
           fill = 'white', color = 'gray30', size = .5) + 
  annotate('segment', x= xanchor, y = yanchor, xend = xanchor + 0.1, yend = yanchor, 
           color = 'gray30') +

  annotate('text', x= xanchor + 0.2, y = ylow,
           hjust = 0, size = 3, label = '25th percentile') +
  annotate('text', x= xanchor + 0.2, y = yanchor,
           hjust = 0, size = 3, label = 'median') +
  annotate('text', x= xanchor + 0.2, y = yhigh,
           hjust = 0, size = 3, label = '75th percentile')
```

### Add Geometric Means

``` r
plt2 + 
   stat_summary(fun = gm_mean, shape = 23, fill = 'gray85')
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/boxplot_final-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/recent_conditons_jitter_box.pdf', device = cairo_pdf, 
       width = 5, height = 4)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

# Trend Graphics

We only have long term data from two beaches – East End and Willard. And
trends are not statistically significant at eh single beach level.
Apparent importance depends on the model uses, which is always
uncomfortable.

For simplicity, we restrict attention to those to near-continuous
records only.

``` r
trend_data <- beach_data %>%
  filter(SiteCode == 'WIL-02' | SiteCode == 'EEB-01') %>%
  filter(! is.na(Bacteria)) %>%
  filter(Year > 2004) %>%
  filter(Reporting_Limit > 5 | is.na(Reporting_Limit))
```

``` r
plt <- ggplot(trend_data, aes(x = Year, y = Bacteria, color = Censored_Flag)) +
  geom_jitter(alpha = .5, height = 0.025, width = 0.2) +
  scale_y_log10() +
  scale_x_continuous(breaks = c(2006, 2010, 2014, 2018)) +
  
  xlab('') +
  ylab('Enterococci (MPN / 100ml)') +
  
  theme_cbep(base_size = 14) +
  theme(panel.border = element_rect(color = 'gray85', fill = NA)) + 

  scale_color_manual(values = cbep_colors(), 
                     labels = c('Observed', 'Below Detection'),
                     name = NULL,
                     guide = guide_legend(ncol  = 2)) +
  # Following adds a little extra space above the plot to make room
  # for a legend, and places the legend a bit closer to the plot
  # than the default "top" position.
  
  theme( plot.margin=unit(c(.25,.25,.5,.25),"inch"),
         legend.position = c(.5,-.2),
         legend.text = element_text(size = 8),
         legend.key.height = unit(10, 'points')) +

  theme(axis.text.x = element_text(size = 9)) +
  facet_wrap(~Beach, nrow = 1)

plt
```

<img src="beaches_graphics_files/figure-gfm/trend_plot-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trend_two_beaches.pdf', device = cairo_pdf, 
       width = 5, height = 4)
```
