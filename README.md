# Криптонит: Классификация эмоций в текстах
Задача для хакатона (проектного практикума) МФТИ ПУСК «Науки о данных», декабрь 2024.
### Задача:
Обучить языковую модель для классификации эмоций в текстах на русском языке. Количество эмоций - 7, при этом один текст может содержать не одну эмоцию, а несколько (multiclass multilabel classification).
## Содержание
1. [Состав команды и роли](#состав-команды-и-роли)
2. [Актуальное состояние проекта](#актуальное-состояние-проекта)
3. [Используемые методы](#используемые-методы)
    * [Предобработка данных](#предобработка-данных)
        - [Расширение датасета](#расширение-датасета)
        - [Аугментация данных](#аугментация-данных)
        - [Обработка эмодзи](#обработка-эмодзи)
        - [Обработка ошибок ASR](#обработка-ошибок-asr)
    * [Архитектура модели](#архитектура-модели)
    * [Гиперпараметры](#гиперпараметры)
4. [Запуск](#запуск)
5. [Основные функции и классы в коде](#основные-функции-и-классы-в-коде)
6. [Выводы](#выводы)
7. [Другие выводы и эксперименты](#другие-выводы-и-эксперименты)
8. [Направления дальнейших исследований](#направления-дальнейших-исследований)

## Состав команды и роли
- Вяткин Роман: Капитан, Scrum master, ML Engineer
- Новиков Валентин: ML Engineer
- Назаров Михаил: ML Engineer
- Яськова Марина: ML Engineer, Data Scientist
- Ихматуллаев Даврон: ML Engineer
- Заславская Вероника: Data Scientist, Data Engineer
- Косачев Дмитрий: MLOps

## Актуальное состояние проекта
Подготовлен ноутбук с решением задачи: [ссылка](./emotion_classification_datasorceres.ipynb).
* Ноутбук подготовлен на основе baseline-решения, предоставленного заказчиком.
* Наш лучший результат по метрике weighted f1-score на тестовой выборке — **0.64**. Метрика рассчитывается тестами [Kaggle-соревнования](https://www.kaggle.com/competitions/cryptonite-hack-sf/).
* По сравнению с baseline-решением мы улучшили результат на 0.1 (baseline-решение показало результат на тесте ~0.54). Это лучший результат в лидерборде Kaggle-соревнования.
* Также мы предлагаем [идеи для дальнейших исследований](#направления-дальнейших-исследований) по улучшению результата.

![kaggle-leaderboard](./img/kaggle.png)




## Используемые методы
### Предобработка данных
#### Расширение датасета
Цель расширения — сгладить дисбаланс классов в тренировочном датасете и стимулировать повышение робастности обучаемой на этих данных модели.

Для расширения датасета использовали датасет https://huggingface.co/datasets/Djacon/ru-izard-emotions. Из него взяли примеры, содержание метки fear, disgust и sadness (24891 строк). Дополнительно в датасет было добавлено еще ~700 строк синтезированных при помощи LLM данных для классов fear и disgust.

* Размер тренировочной выборки, предоставленной заказчиком: 43410 строк.
* Размер тренировочной выборки после расширения датасета: 69973 строк.

Baseline-датасет:
![img baseline dataset](./img/dataset_baseline.png)

Датасет после расширения:
![img ext dataset](./img/dataset_extended.png)


#### Обработка эмодзи
Реализована при помощи библиотеки [emoji](https://pypi.org/project/emoji/).

Функция вида emoji.demojize("👍", language='ru') переводит иконки эмодзи в текстовое представление на русском языке.





### Архитектура модели
Подход, который мы выбрали для решения задачи — это fine-tuning предобученных моделей, размещенных в репозитории Huggingface.

* В рамках исследования мы протестировали ряд BERT-like архитектур моделей (BERT, RoBERTa, DistilBERT и др.), а также некоторые модели с архитектурами других типов.
* Текущий лучший результат показала модель с архитектурой RoBERTA.

Условная схема инференса модели приведена ниже на картинке:
![img architecture](./img/arch.png)

### Гиперпараметры
Мы экспериментировали со значениями следующих гиперпараметров:
* Количество эпох
* Величина шага оптимизатора
* Сила регуляризации оптимизатора
* Тип оптимизатора
* Балансировка (weight) в лосс-функции
* Количество классификационных выходных слоев
* Тип активационных функций
* Тип нормализации
* Размер скрытого слоя
* Величина порога уверенности для фильтрации предсказаний
* С увеличением числа линейных слоев в «голове» модели мы меняем уровень Dropout, который их разделяет.






## Запуск
Код решения содержится в [ноутбуке](./emotion_classification_datasorceres.ipynb), представляющем собой модифицированное baseline-решение. Для воспроизведения результатов достаточно выполнить последовательный запуск всех ячеек.

Ноутбук содержит пути до файлов с данными, при запуске их потребуется заменить на актуальные.

Расширенный тренировочный датасет можно скачать [здесь](./data/train_plus_djacon.csv). Валидационную выборку мы оставили без изменений.

Дообученную нами модель можно загрузить по [ссылке](https://disk.yandex.ru/d/42y7GiGesJGhQw).

Для обучения мы использовали предобученную модель ShakurovR-knv_model_v2, которая была размещена на Huggingface, однако в середине хакатона была удалена автором. Сохраненную версию этой модели можно скачать [здесь](https://disk.yandex.ru/d/Hk0BKtcJin8UAQ).

## Основные функции и классы в коде
* seed_everything: функция для фиксации всех используемых в коде генераторов случайных величин. Обеспечивает воспроизводимость.
* get_alphabet: функция для выведения всех уникальных символов в наборе данных.
* plot_histogram: функция для построения гистограммы распределения классов (эмоций) в датасете
* plot_combined_emotions: функция для построения распределения попарной встречаемости эмоций в примерах датасета
* class Model: класс оборачивает предобученную модель и добавляет к ней классификационную "голову"
* class EmotionDataset: класс для подготовки датасета к обучению модели
* train: функция, содержащая цикл обучения модели
* validation: функция для получения предсказаний модели. Используется для валидации во время цикла обучения, а также для получения предсказаний на тестовой выборке

## Выводы
Нам удалось достичь метрики weighted F1-score на уровне 0.64. Улучшение достигнуто за счет комплексного подхода к обработке данных, выбора архитектуры модели и экспериментов с гиперпараметрами.
*	Увеличение тренировочной выборки с 43,410 строк до 69,973 строк (на 61%) помогло сгладить дисбаланс классов и повысить робастность модели.
*	Использование внешнего датасета с метками эмоций *fear*, *disgust*, *sadness* и синтезированных данных для классов fear и disgust позволило улучшить представление редких классов.
*	Лучшая производительность на данный момент достигнута с помощью модели на основе RoBERTa.
*	Оптимальный размер скрытого слоя находится примерно в промежутке 750–800. Выход размера скрытого слоя за пределы этого промежутка как в большую, так и в меньшую сторону, уменьшает значение метрики weighted F1-score.
*	Добавление второго линейного слоя в классификационную «голову» позволило улучшить метрики.
*	Добавление Dropout-слоя между двумя линейными слоями классификационной «головы» позволило улучшить метрики. Оптимальным с точки зрения увеличения weighted F1-score на тестовой выборке оказался уровень Dropout в районе 0.85.
*	Снижение порога уверенности для фильтрации предсказаний с 0.5 (в baseline-решении) до 0.354 позволило увеличить weighted F1-score на тестовой выборке.



## Другие выводы и эксперименты
Мы пробовали добавлять обработку эмодзи (преобразование их в текст) для получения дополнительной информации, однако к улучшению метрик это не привело. Причина в том, что в тестовой выборке эмодзи отсутствуют. Поэтому, несмотря на то, что обработка эмодзи могла бы повлиять на общую робастность модели, мы исключили ее из решения, поскольку она не была актуальна для задачи хакатона.

Мы также провели ряд других экспериментов, которые не привели к значимому улучшению метрик:
* Балансировка весов классов в лосс-функции.
* Замена sheduler для warmup на косинусный.
* Замена функций активации на SiLu, GeLu, Normalize, Dropout.
* Добавление нелинейности в классификационную голову. Эксперименты проводились с разными типами активационных слоев.
* Увеличение числа классификационных голов (от 2 до 7 классификационных голов, верхняя граница — по количеству классов). Тестировались «головы» с различным количеством нейронов и различными активациями.
* Добавление мета-слоя, конкатенировавшего выходы классификационных голов.
* Ручное добавление pooling-а на выход предобученных тех моделей, в архитектуре которых его не было. Тестировались варианты: average pooling, max pooling, конкатенация двух видов пулинга.




## Направления дальнейших исследований

#### Аугментация данных
Для повышения устойчивости модели к "испорченным" ASR-данным мы предлагаем протестировать применение к данным аугментации:
* Перевод на английский язык и обратно на русский с помощью моделей семейcтва [Helsinki-NLP](https://huggingface.co/Helsinki-NLP)
* Замена части слов случайным синонимом из корпуса wordnet (NLTK).

#### Обработка ошибок ASR
Методом визуального анализа в данных были выявлены некоторые часто встречающие ошибки, генерируемые ASR-моделью. 
Создание словаря из таких ошибок, соотнесение их с корректным написанием слов и произведение в тексте замены может привести к повышению метрик.

Например: "ыетс" —> "это", "ужеж" —> "уже".

Альтернативой может послужить добавление в пайплайн инференса алгоритма для автоматического исправления опечаток (например, Яндекс.Спеллер).

#### Обработка эмодзи и эмотиконов
В случае обобщения модели для работы над текстовыми данными из интернета имеет смысл повторно протестировать обработку эмодзи, а также обработку эмотиконов (смайликов, изображаемых с помощью текстовых символов и спецсимволов). Перевод их значения в текстовый формат может предоставить модели дополнительные данные о настроении высказывания.

#### Очистка и нормализация данных для инференса
В пайплайн инференса стоит добавить блок по очистке и нормализации данных, аналогичный блоку, применяемому к тренировочным и валидационным данным.


