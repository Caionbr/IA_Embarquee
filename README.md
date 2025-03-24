# Projet IA_Embarquee

**Participants:**  
Caio Vinícius Rodrigues Nobre  
Murilo Mattos Muller

**Institution:**  
École Nationale Supérieure des Mines de Saint-Étienne (EMSE-ISMIN)  

---

## Folder structure

  - *Communication_STM32*: contains the python script (“*com.py*”) to test communication with the STM32 board and displays the accuracy (in the terminal). The folder also contains the inputs used, as well as an example of the output that can be viewed in the terminal (“*test_acccuracy*”).
  - *MNIST*: network used to learn how to use cubeMX/IDE, provided by the professor at ecampus.
  - *Machine_Failure*: analysis and validation of the project in cubeIDE with the google colab “*TP_IA_EMBARQUEE*” completed.
  - *Notebook*: contains the google colab file of the trained “*AI4I 2020 Predictive Maintenance Dataset*” model.
  - *ReportCubeMX*: contains the files generated by cubeMX after analysis and validation of the programmed model.
  - *archives_from_colab*: contains the files generated in google colab (“*model_test.h5*”, “*X_test_labels.npy*” and “*Y_test_labels.npy*”) that will be used for analysis and validation in CubeMX. 
---

## Part 1 – Model development on Google Colab

### Dataset Used

The model was trained using the "AI4I 2020 Predictive Maintenance Dataset", available in .csv format. This dataset contains information on different types of faults in industrial machinery, as well as operating variables such as temperature, torque and tool wear.

---

### Objective

The aim is to develop an AI model to identify whether a fault has occurred, and if so, what type. The model works with a multi-class classification and can identify one of the 4 types of faults or whether there was no fault:

1. No Failure 
2. TWF (Tool Wear Failure)
3. HDF (Heat Dissipation Failure)
4. PWF (Power Failure)
5. OSF (Overstrain Failure)

---

### Data pre-processing
Initially, we analyzed and processed the data provided, following the different techniques applied:

#### One-Hot Coding
The Type column initially contained the strings M, L and H, so it was converted to one-hot variables, allowing the network to interpret this categorical information.

```python
df = pd.get_dummies(df, columns=['Type'], drop_first=False)   
```

#### Removing simultaneous faults
As the aim is to classify a single fault per sample, samples with multiple faults were removed to avoid ambiguities in the training.

```python
failure_types = ['TWF', 'HDF', 'PWF', 'OSF']
df['sum_failures'] = df[failure_types].sum(axis=1)
df = df[(df['Machine failure'] == 0) | ((df['Machine failure'] == 1) & (df['sum_failures'] == 1))]
df.drop(columns='sum_failures', inplace=True)
```

#### Random fault removal (RNF)
Random faults (RNF) were also removed from the dataset, as there is no way to predict a random fault.

```python
Y = df[['No failure', 'TWF', 'HDF', 'PWF', 'OSF']]
```

#### Creating the target vector(Y)
A target vector was created with 5 columns, representing the 5 possible classes. Each row contains a single value equal to 1, representing the true class of the sample (one-hot encoding format).

```python
df['No failure'] = (df['Machine failure'] == 0).astype(int)
Y = df[['No failure', 'TWF', 'HDF', 'PWF', 'OSF']].copy()
Y = Y.apply(lambda row: (row == 1).astype(int), axis=1)
```

#### Data normalization
The input features were normalized with MinMaxScaler to ensure that all values are on the same scale and to make it easier for the network to learn.

```python
X = df.drop(columns=['UDI', 'Product ID', 'Machine failure', 'TWF', 'HDF', 'PWF', 'OSF', 'RNF', 'No failure'])
scaler = MinMaxScaler()
X_scaled = scaler.fit_transform(X)
X = pd.DataFrame(X_scaled, columns=X.columns)
```

#### Balancing Classes
The dataset shows a large imbalance between the samples with and without failures, with only 3.39 % of failures. This has caused the model to tend to predict the non-failed label in most cases, because even if it makes a mistake, the large amount of data from the non-failed label means that the loss and accuracy are apparently good. To correct this:

The undersampling method was used with RandomUnderSampler, reducing the number of samples from the majority class (No Failure) so that there is a more balanced proportion with the others.

```python
undersampler = RandomUnderSampler(sampling_strategy=0.3, random_state=42)
X_resampled, y_no_failure = undersampler.fit_resample(X, Y['No failure'])
selected_indices = y_no_failure.index
Y_resampled = Y.iloc[selected_indices].copy()
Y_resampled = Y_resampled.apply(lambda row: (row == 1).astype(int), axis=1)
```

A weight was assigned to each type of fault, even after reducing the majority class there are differences between the minority classes, the TWF class for example has fewer samples compared to the other faults, so it receives a higher weight.

```python
y_integers = Y_resampled.values.argmax(axis=1)
classes = np.unique(y_integers)
class_weights = compute_class_weight(class_weight='balanced', classes=classes, y=y_integers)
class_weight_dict = {i: weight for i, weight in zip(classes, class_weights)}
```

#### Division into Training and Testing
The data was divided into:
1. 70% for training
2. 30% for testing

The division was made in a stratified way based on the true class (argmax of each Y vector), ensuring that the proportion between classes is preserved in both sets.

```python
X_train_under, X_test_under, Y_train_under, Y_test_under = train_test_split(
    X_resampled, Y_resampled, test_size=0.2, random_state=42,
    stratify=Y_resampled.values.argmax(axis=1)
)
```

---

### Neural Network Architecture
The neural network was developed using the TensorFlow/Keras library, focusing on a lightweight and efficient design, ideal for embedded devices such as STM32 family microcontrollers. The architecture consists of:


#### Hidden layers:


1. Dense layers for learning complex representations.
2. Batch Normalization to stabilize and speed up training.
3. LeakyReLU as an activation function, avoiding the problem of "dead neurons".
4. Dropout (between 20% and 40%) to reduce overfitting.
5. L2 regularization to penalize very high weights, favoring generalization.

#### Output layer:

1. With 5 neurons (representing the classes: No failure, TWF, HDF, PWF, OSF).
2. Softmax activation function, returning the probabilities for each class.

```python
model = Sequential([
    Dense(64, kernel_regularizer=l2(0.01), input_shape=(X_train_under.shape[1],)),
    BatchNormalization(),
    LeakyReLU(),
    Dropout(0.2),

    Dense(32, kernel_regularizer=l2(0.01)),
    LeakyReLU(),
    Dropout(0.2),

    Dense(5, activation='softmax')  
])

model.compile(
    optimizer=AdamW(learning_rate=0.001, weight_decay=0.02),
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

model.summary()
```

---

### Model training
Training was carried out with a focus on high accuracy and good generalization, using the following components:

1. Loss function:

categorical_crossentropy, ideal for multiclass classification with one-hot coding.

2. Optimization:

AdamW (a version of the Adam optimizer with weight decay), which improves overfitting control without losing convergence speed.

3. Callbacks used:

EarlyStopping: stops training if validation stops improving, avoiding overfitting.
ReduceLROnPlateau: automatically reduces the learning rate if performance stagnates.

4. Treatment of class imbalance:

Class weights were calculated automatically with compute_class_weight, reinforcing the importance of classes

```python
early_stopping = EarlyStopping(monitor='val_loss', patience=5, restore_best_weights=True, verbose=1)
reduce_lr = ReduceLROnPlateau(monitor='val_loss', factor=0.5, patience=4, min_lr=1e-6, verbose=1)

history = model.fit(
    X_train_under,
    Y_train_under,
    validation_data=(X_test_under, Y_test_under),
    epochs=50,
    batch_size=64,
    class_weight=class_weight_dict,
    callbacks=[early_stopping, reduce_lr],
    verbose=1
)
```

---

### Evaluation
During the test, the model was evaluated with the following metrics:

1. General accuracy
2. Precision, Recall and F1-Score by class
3. Confusion matrix

These metrics showed that, despite the imbalance, the model manages to learn to identify specific faults with good performance, especially after balancing the data.

```python
Y_pred_probs = model.predict(X_test_under)
Y_pred = Y_pred_probs.argmax(axis=1)
Y_true = Y_test_under.values.argmax(axis=1)

labels = ['No failure', 'TWF', 'HDF', 'PWF', 'OSF']
print(classification_report(Y_true, Y_pred, target_names=labels))

cm = confusion_matrix(Y_true, Y_pred)
cm_percent = cm.astype('float') / cm.sum(axis=1, keepdims=True) * 100

plt.figure(figsize=(8, 6))
sns.heatmap(cm_percent, annot=True, fmt=".2f", cmap="Reds",
            xticklabels=labels, yticklabels=labels)
plt.title("Normalized Confusion Matrix (%)")
plt.xlabel("Predict class")
plt.ylabel("Real class")
plt.show()
```

Here are the results obtained after training and testing the model. It is worth noting that when training and testing the model again, the results will not be the same due to the randomness of the dataset separation, but the performance does not vary much.

| Classe       | Precision | Recall | F1-Score | Suporte |
|--------------|-----------|--------|----------|---------|
| No failure   | 0.98      | 0.88   | 0.93     | 205     |
| TWF          | 0.24      | 0.62   | 0.34     | 8       |
| HDF          | 0.95      | 0.90   | 0.93     | 21      |
| PWF          | 0.80      | 1.00   | 0.89     | 16      |
| OSF          | 0.71      | 0.94   | 0.81     | 16      |


**Overall accuracy:** 0.89  
**Mmacro avg:** Precision = 0.74, Recall = 0.87, F1-score = 0.78  
**Weighted avg:** Precision = 0.93, Recall = 0.89, F1-score = 0.90

---

### Model export
The final model was converted into an STM32-compatible format using STM32Cube.AI, and uploaded to the board for real-time execution, with UART communication for sending and receiving data.

```python
np.save('X_test_labels.npy', X_test_under)
Y_test_under = Y_test_under.astype(np.float32)
np.save('Y_test_labels.npy', Y_test_under)
model.save('model_test.h5')
```

---

## Part 2 - CubeMX/IDE development

---

### Objective

Analyze and validate the “*AI4I 2020 Predictive Maintenance Dataset*” model trained on Google Colab in CubeMX, as well as program UART communication on the STM32L4R9AI development board in CubeIDE. Finally, test the communication with a python script, in VsCode, that returns the accuracy of the model embedded on the board.

---

### Steps taken

1. **Use of CubeMX**
   - Configuration of the "*X-CUBE-AI*" in the categorie of "*Middleware and Software Packs*";
   - Selection of the core in the “*Artificial Intelligence X-CUBE-AI*” tab and the “*Application Template*” in the “*Device Application*” tab;
   - After, going to the “*X-CUBE-AI*” settings, we added a new network called Machine_Failures. Then, using the “*Keras*” model, we select the files generated by the colab: “*model_test.h5*” by clicking browse to load it and “*X_test_labels.npy*” and “*Y_test_labels.npy*” for Validation Inputs and Validation outputs, respectively;
   - Now that the files have been uploaded, we analyze and validate them on the desktop (in this sequence), thus obtaining a good response from the model as can be seen in the files from the ReportCubeMX folder;
   - Finally, go to the “*Project Manager*” tab to configure and start programming a project in CubeIDE.
2. **Use of CubeIDE**
   - Configuration of the “*main.c*” as instructed by the professor, commenting out a few lines in the “* Initialize all configured peripherals *” section:
       ```C
       /* Initialize all configured peripherals */
       MX_GPIO_Init();
       MX_FMC_Init();
       MX_I2C1_Init();
       MX_SAI1_Init();
       //MX_SDMMC1_SD_Init();
       MX_SPI2_Init();
       MX_USART2_UART_Init();
       MX_USART3_UART_Init();
       //MX_USB_OTG_FS_PCD_Init();
       MX_X_CUBE_AI_Init();
       
   -  After that, the “*app_x-cube-ai.c*” file was configured as instructed by the ecampus files. However, specific modifications were made to ensure correct communication with the python script. The functions:
     ```C
     int acquire_and_process_data(ai_i8* data[])
     int post_process(ai_i8* data[])
     ```
    - They have been changed to:
     ```C
       int acquire_and_process_data(uint8_t* data)
       int post_process(uint8_t* data)
     ```
    - And the looping structure:
    ```C
      do {
      res = acquire_and_process_data(in_data);
      if (res == 0)
          res = ai_run();
      if (res == 0)
          res = post_process(out_data);
      } while (res == 0);
     ```  
    - It was changed to:
  
    ```C
      while (1) {
      res = acquire_and_process_data(in_data);
      if (res != 0) continue;
  
      res = ai_run();
      if (res != 0) continue;
  
      res = post_process(out_data);
      if (res != 0) continue;
      }
     ```  
3. **Use of VsCode for python script**
   - For the python script, we used the same one provided by ecampus, changing only the directory of the file for the training tests provided by our model. The only change was:
   ```python
    if __name__ == '__main__':
        X_test = np.load(r"D:\EMSE\IA EMBARQUEE\RepositorioRemoto\IA_Embarquee\Communication_STM32\X_test_labels.npy")
        Y_test = np.load(r"D:\EMSE\IA EMBARQUEE\RepositorioRemoto\IA_Embarquee\Communication_STM32\Y_test_labels.npy")
    
    print(X_test.shape)
    print(Y_test.shape)

   ```
   - The results obtained were satisfactory, even though they still contain many examples of the “No failure” type. An example of the terminal output has been saved in the file “*test_accuracy.txt*”. 
  


