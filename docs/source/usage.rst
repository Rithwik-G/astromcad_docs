Usage
=====

.. _installation:

Installation
------------

To use astromcad, first install it using pip:

.. code-block:: console

    (.venv) $ pip install astromcad

Using Pretrained Models
-----------------------

.. code-block:: python
   import os
   from utils.load_data import get_data
   from astromcad import build_model, train
        
   X_train, X_val, X_test, host_gal_train, host_gal_val, host_gal_test, y_train, y_val, y_test, class_weights, ntimesteps, x_data_anom, host_gal_anom, y_data_anom = get_data()
   y_class = np.array([classes[np.argmax(i)] for i in y_test])

   Detect.init(host=True)
   os.makedirs("scores", exist_ok=True)

   path = "scores/norm_scores.csv"
   anomaly_detector.generate_score_csv([X_test, host_gal_test], path, y_class)
   print("Normal Class Scores Generated")

   path = "scores/anom_scores.csv"
   anomaly_detector.generate_score_csv([x_data_anom, host_gal_anom], path, y_data_anom)
   print("Anom Class Scores Generated")

The score files generated can be used directly to generate plots (see :ref:`plots`).
   

Training Custom Models
----------------------

Step 1: Build a Classifier
~~~~~~~~~~~~~~~~~~~~~~~~~~

To build an anomaly detector on your data, first train a classifier. 

.. code-block:: python

   from utils.load_data import get_data
   from astromcad import build_model, train
        
   X_train, X_val, X_test, host_gal_train, host_gal_val, host_gal_test, y_train, y_val, y_test, class_weights, ntimesteps, x_data_anom, host_gal_anom, y_data_anom = get_data()

   print("Building Model")
   # model = build_model(100, ntimesteps, y_train.shape[1], contextual=0, n_features = 4)
   model = build_model(100, ntimesteps, y_train.shape[1], contextual=2, n_features = 4)
   print(model.summary())
   
   print("Training")
   # history = train(model, X_train, y_train, X_val, y_val, class_weights, epochs=40)
   history = train_contextual(model, X_train, host_gal_train, y_train, X_val, host_gal_val, y_val, class_weights, epochs=40)
   
   # model.save("Models/NoHostClassifier.h5")
   model.save("Models/HostClassifier.h5")
   
   print("Model Saved")


Step 2: Initialize MCIF
~~~~~~~~~~~~~~~~~~~~~~~

Then, initialize the MCIF anomaly detector with the trained classifier.

.. code-block:: python

   from astromcad import mcad, mcif
   from tensorflow.keras.models import load_model

   y_single = np.array([np.argmax(i) for i in y_train])


   model = load_model("../Models/HostClassifier.h5")

   anomaly_detector = mcad(model, 'latent', ['lc', 'host'])
   anomaly_detector.create_encoder()
   anomaly_detector.init_mcif([X_train, host_gal_train], y_single)

   save('../Models/HostMCIF.pickle', anomaly_detector.mcif)

Step 3: Detect Anomalies
~~~~~~~~~~~~~~~~~~~~~~~~

Finally, use the MCIF anomaly detector to detect anomalies in your data.

.. code-block:: python
   import os
   
   y_class = np.array([classes[np.argmax(i)] for i in y_test])

   os.makedirs("scores", exist_ok=True)

   path = "scores/norm_scores.csv"
   anomaly_detector.generate_score_csv([X_test, host_gal_test], path, y_class)
   print("Normal Class Scores Generated")

   path = "scores/anom_scores.csv"
   anomaly_detector.generate_score_csv([x_data_anom, host_gal_anom], path, y_data_anom)
   print("Anom Class Scores Generated")

The score files generated can be used directly to generate plots (see :ref:`plots`).

.. _plots:

Plotting Results
----------------

.. code-block:: python

   from utils.figures import *
   import pandas as pd
   import random

   norm_results = pd.read_csv("scores/norm_scores.csv")
   anom_results = pd.read_csv("scores/anom_scores.csv")

   plot_recall(list(norm_results['score']), random.sample(list(anom_results['score']), 100))
   plt.savefig("../figures/recall.pdf", bbox_inches='tight')
   plt.show()
   print("Generated Recall Plot")

   median_score(norm_results, anom_results)
   plt.savefig("../figures/median_score.pdf", bbox_inches='tight')
   plt.show()
   print("Generated Median Score Plot")

   distribution(norm_results, anom_results)
   plt.savefig("../figures/anomaly_distribution.pdf", bbox_inches='tight')
   plt.show()
   print("Generated Anomaly Distribution Plot")
