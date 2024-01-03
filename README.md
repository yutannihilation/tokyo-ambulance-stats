# 東京都の救急車出動件数の推移

## データ

### データソース

データは「救急活動の現況」として東京消防庁ホームページで公開されているが、5年分しか掲載されず、過去分は消されてしまうらしい。2016年、2017年はInternet
ArchiveのWeyback Machineに残っていたのでそこから値を拾う。

- 2018～2022:
  https://www.tfd.metro.tokyo.lg.jp/hp-kyuukanka/katudojitai/index.html

- 2017:
  https://web.archive.org/web/20220308120712/https://www.tfd.metro.tokyo.lg.jp/hp-kyuukanka/katudojitai/29.pdf

- 2016:
  https://web.archive.org/web/20220308120657/https://www.tfd.metro.tokyo.lg.jp/hp-kyuukanka/katudojitai/28.pdf

### データ整形

#### 2018～2022

2018年以降は図表ごとのExcel、PDFが提供されており、今回は「図表2-1-15
月別出場件数」を使う。
図表番号は年度が替わっても固定らしく、URLも年号部分が変わるだけ。

``` r
library(glue)

data_dir <- here::here("data")
dir.create(data_dir, showWarnings = FALSE)

download_xls <- function(year) {
  destfile <- glue("{data_dir}/{year}.xlsx")
  
  # 再実行しても大丈夫なようにしておく
  if (file.exists(destfile)) {
    rlang::inform(glue("skip {year}...\n\n"))
    return(invisible(NULL))
  }
  
  url <- glue("https://www.tfd.metro.tokyo.lg.jp/hp-kyuukanka/katudojitai/data/Excel/{year}_2-1-15.xlsx")
  curl::curl_download(url, destfile)
}

years <- c("H30", "R1", "R2", "R3", "R4")
purrr::walk(years, download_xls)
```

    skip H30...

    skip R1...

    skip R2...

    skip R3...

    skip R4...

``` r
excel_files <- list.files(data_dir, full.names = TRUE)
basename(excel_files)
```

    [1] "H30.xlsx" "R1.xlsx"  "R2.xlsx"  "R3.xlsx"  "R4.xlsx" 

``` r
library(dplyr, warn.conflicts = FALSE)

western_years <- paste0(2018:2022, "年")
names(western_years) <- years

d <- excel_files |> 
  purrr::map(\(x) {
    result <- readxl::read_xlsx(x, skip = 2, col_types = c("text", "numeric", "skip"))
    colnames(result) <- c("month", "count")
    result |> 
      filter(month != "合計") |> 
      mutate(
        year = western_years[tools::file_path_sans_ext(basename(x))],
        month = readr::parse_number(month),
        .before = month
      )
  }) |> 
  purrr::list_rbind()
```

#### 2016、2017

(TODO)

## 年間出場件数

``` r
d |> 
  group_by(year) |> 
  summarise(count = sum(count))
```

    # A tibble: 5 × 2
      year    count
      <chr>   <dbl>
    1 2018年 818062
    2 2019年 825929
    3 2020年 720965
    4 2021年 743703
    5 2022年 872075

## プロット

``` r
library(ggplot2)

ggplot(d, aes(month, count, colour = year)) +
  geom_line() +
  scale_color_viridis_d(option = "A", direction = -1, end = 0.9) +
  scale_x_continuous(breaks = 1:12) +
  scale_y_continuous(labels = scales::label_comma(accuracy = 1)) +
  labs(x = "月", y = NULL, title = "東京都の救急出場件数",
       caption = "出典：東京都消防庁「救急活動の現況」") +
  theme_minimal() +
  theme(
    text = element_text(size = 25)
  ) +
  gghighlight::gghighlight(label_key = year, line_label_type = "sec_axis")
```

![](README_files/figure-commonmark/plot-1.png)
