
# Mở đầu

Những gói tạo nên tidymodels không làm nhiệm vụ thực hiện các mô hình thống kê. Mà thay vào đó, họ tập trung vào làm những việc xung quanh việc xây dựng mô hình, làm cho mô hình dễ hiểu hơn. 

Xây dựng mô hình có nhiều bước nhỏ, mỗi gói trong hệ sinh thái này làm 1 hoặc vài phần trong các bước xây dựng mô hình:

- rsample – Different types of re-samples
- recipes – Transformations for model data pre-processing
- parsnip – A common interface for model creation
- yardstick – Measure model performance

Gói tidymodels không hoặc hiện tại chưa làm với dự báo chuỗi thời gian 

# Minh họa các bước thực hiện bằng dữ liệu iris

## Pre-Process

Hầu hết việc biến đổi dữ liệu sẽ sử dụng gói dplyr. Các packages khác của hệ sinh thái tidyverse sử dụng đối với từng mô hình chuyên biệt hoặc phức tạp.

## Data Sampling

Hàm initial_split() sử dụng để chia dữ liệu thành tập train và tập test. Mặc định, tỷ lệ 3:1 cho tập train và tập test. 

```{r}
library(tidymodels)
iris_split <- initial_split(iris, prop = 0.6)
iris_split
```

Để lấy dữ liệu trainning dùng hàm trainning()
```{r}
iris_split %>%
  training() %>%
  glimpse()
```

## Data pre-processing

recipes package cung cấp giao diện làm việc đối với tiền xử lý dữ liệu. Gói này cung cấp các hàm theo từng bước chế biến y hệt như công việc nấu nướng :D.

- recipe(): hàm khởi tạo, y hệt như ggplot()
- prep(): Thực hiện biến đổi dữ liệu. Mỗi cách biến đổi dữ liệu sẽ được thực hiện bởi 1 hàm bắt đầu bằng tiền tố step_. Ví dụ:

  - step_corr(): loại những biến có tương quan cao

  - step_center(): Chuẩn hóa biến numeric về trung bình bằng 0

  - step_scale(): Chuẩn hóa biến numeric về trung bình bằng 0, phương sai bằng 1

- Một số hàm khác dùng để gói các biến theo nhóm như: all_outcomes() và all_predictors() dùng để gọi tất cả các biến outcomes hoặc các biến predictors

Ví dự sử dụng các hàm recipe(), prep(), các hàm step_ để tạo ra recipe object. Các bước thực hiện sẽ được mô tả một cách chi tiết  

```{r}
iris_recipe <- training(iris_split) %>%
  recipe(Species ~.) %>%
  step_corr(all_predictors()) %>%
  step_center(all_predictors(), -all_outcomes()) %>%
  step_scale(all_predictors(), -all_outcomes()) %>%
  prep()

iris_recipe
```

Các bước biến đổi dữ liệu đối với tập training sẽ được thực hiện giống hệt với tập testing. Để thực hiện biến đổi sử dụng hàm bake().

Chú ý, hàm testing() sử dụng để gọi dữ liệu testing

```{r}
iris_testing <- iris_recipe %>%
  bake(testing(iris_split)) 

glimpse(iris_testing)
```

Đối với dữ liệu training, để lặp lại các bước như trên sẽ là thừa, ta chỉ cần sử dụng hàm juice để trích xuất dữ liệu

```{r}
iris_training <- juice(iris_recipe) 

# Dùng juice tương đương với các bước dưới
# iris_training2 <- iris_recipe %>%
#   bake(training(iris_split)) 
# identical(iris_training, iris_training2)

glimpse(iris_training)
```

## Model Training

Trên R, có rất nhiều packages làm cùng nhiệm vụ như nhau với cùng loại mô hình. Mỗi package có 1 số tên agruments riêng, làm cho người dùng khó nhớ. Vì dụ: ranger và randomForest cùng làm nhiệm vụ fit random Forest model. Để khai báo number of tree, ranger dùng num.trees, randomForest dùng ntree. Không dễ để nhớ khi thay đổi package.

Thay vì việc thay thế modeling package, tidymodels thay thể 1 chút trong giao diện. Hay nói cách khác là tidymodels cung cấp agrument giống nhau cho cùng loại mô hình. 

Ví dụ: sử dụng hàm rand_forest() trong tidymodels. Thay vì đổi hàm và đổi gói, chỉ việc thay đổi set_engine(). Và sử dụng hàm fit() để khai báo dạng mô hình.

```{r}
iris_ranger <- rand_forest(trees = 100, mode = "classification") %>%
  set_engine("ranger") %>%
  fit(Species ~ ., data = iris_training)

```

```{r}
iris_rf <-  rand_forest(trees = 100, mode = "classification") %>%
  set_engine("randomForest") %>%
  fit(Species ~ ., data = iris_training)
```

Các agruments định nghĩa như trên sẽ làm cho người dùng linh động và dễ dàng hơn khi chuyển đổi qua lại.

Hơn nữa, hàm predict() của parsnip trả về tibble. Mặc định, predict variable đặt tên **.pred_class**.

```{r}
predict(iris_ranger, iris_testing)
```

Rất dễ dàng khi sử dụng hàm dplyr’s bind_cols() để nối vào dữ liệu testing đã được baked
```{r}
iris_ranger %>%
  predict(iris_testing) %>%
  bind_cols(iris_testing) %>%
  glimpse()
```

## Model Validation

Sử dụng hàm metrics() để đo lượng performance của mô hình. Hàm này rất hay ở chỗ **tự động** chọn metrics phù hợp với loại mô hình. Hàm này cần phải khai báo 2 agruments là *truth* và *estimate*. 

```{r}
iris_ranger %>%
  predict(iris_testing) %>%
  bind_cols(iris_testing) %>%
  metrics(truth = Species, estimate = .pred_class)
```

```{r}
iris_rf %>%
  predict(iris_testing) %>%
  bind_cols(iris_testing) %>%
  metrics(truth = Species, estimate = .pred_class)
```

Xác suất dự báo đối với từng class

Thêm type = "prob" vào hàm predict để có được xác suất dự báo với từng class


```{r}
iris_ranger %>%
  predict(iris_testing, type = "prob") %>%
  glimpse()
```

Cũng rất dễ dàng để nối vào dữ liệu testing

```{r}
iris_probs <- iris_ranger %>%
  predict(iris_testing, type = "prob") %>%
  bind_cols(iris_testing)

glimpse(iris_probs)
```

## Curve methods

```{r}
iris_probs%>%
  gain_curve(Species, .pred_setosa:.pred_virginica) %>%
  glimpse()
```

Để hình ảnh hóa kết quả dự báo sử dụng hàm autoplot()

```{r}
iris_probs%>%
  gain_curve(Species, .pred_setosa:.pred_virginica) %>%
  autoplot()
```

```{r}
iris_probs%>%
  roc_curve(Species, .pred_setosa:.pred_virginica) %>%
  autoplot()
```

- Kết hợp xác suất dự báo cho mỗi giá trị, kết quả dự báo (không dùng xác suất) và giá trị thực tế. Sử dụng kết hợp với hàm dplyr’s select() cho ra kết quả easy to read


```{r}
predict(iris_ranger, iris_testing, type = "prob") %>%
  bind_cols(predict(iris_ranger, iris_testing)) %>%
  bind_cols(select(iris_testing, Species)) %>%
  glimpse()
```

```{r}
predict(iris_ranger, iris_testing, type = "prob") %>%
  bind_cols(predict(iris_ranger, iris_testing)) %>%
  bind_cols(select(iris_testing, Species)) %>%
  metrics(truth = Species, .pred_setosa:.pred_virginica, estimate = .pred_class)
```

Nguồn: https://www.r-bloggers.com/a-gentle-introduction-to-tidymodels/

