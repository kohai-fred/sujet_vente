# Questions et analyses statistiques

1. Nombre total de transactions :

   - Compter le nombre total de documents dans la collection sales.

     ```python
      db.sales.countDocuments()
      # ou
      db.sales.count()

      # ex= 20
     ```

2. Ventes totales par jour :

   - Donnez le total des ventes quotidiennes.

     ```python
     db.sales.aggregate([
         {
           $group: {
             _id: { $dateToString: { format: "%Y-%m-%d", date: "$date" } },
             totalSales: { $sum: "$total_amount" }
           }
         },
       {
         $sort: { _id: 1 }
       }
     ])

     # ex= [{ _id: '2023-07-02', totalSales: 19.98 },...]
     ```

     - On groupe dabord par date (dans un format lisible) puis on fait le total des ventes. Enfin on tri du plus récent
       au plus ancien.

3. Ventes totales par produit :

   - Donnez le total des ventes pour chaque produit.

   ```python
    db.sales.aggregate([
     {
       $group: {
         _id: "$product_id",
         totalSales: { $sum: "$total_amount" }
       }
     },
     {
       $sort: { totalSales: -1 }
     }
   ])

   # ex= [{ _id: 'P67910', totalSales: 133.98 },...]
   ```

   - On groupe par id de produit puis on fait la somme des ventes.

4. Top 5 des produits les plus vendus :

   - Identifiez les 5 produits avec le plus grand nombre de transactions.

   ```python
    db.sales.aggregate([
      {
        $group: {
          _id: "$product_id",
          transactionCount: { $sum: 1 },
        },
      },
      {
        $sort: { transactionCount: -1 },
      },
      {
        $limit: 5,
      },
    ]);

   # ex = [
    #  { _id: 'P67917', transactionCount: 1 },
    #  { _id: 'P67909', transactionCount: 1 },
    #  { _id: 'P67911', transactionCount: 1 },
    #  { _id: 'P67910', transactionCount: 1 },
    #  { _id: 'P67906', transactionCount: 1 }
    #]

   db.sales.aggregate([
      {
        $group: {
          _id: "$product_id"
        }
      },
      {
        $count: "distinctProducts"
      }
    ])
   # Dommage chaque produit n'a qu'une seule transaction...
    # ex= [ { distinctProducts: 20 } ]
   ```

5. Revenu moyen par transaction :

   - Calculez le revenu moyen par transaction.

   ```python
    db.sales.aggregate([
      {
        $group: {
          _id: null,
          averageRevenuePerTransaction: { $avg: "$total_amount" }
        }
      }
    ])

   # ex= [ { averageRevenuePerTransaction: 58.4785 } ]
   ```

6. Nombre de clients uniques :

   - Comptez le nombre de clients uniques ayant effectué au moins une transaction.

   ```python
   db.sales.aggregate([
     {
       $match: {
         quantity: { $gt: 0 },
       },
     },
     {
       $group: {
         _id: "$customer_id",
       },
     },
     {
       $count: "uniqueCustomers",
     },
   ])

   # ex= [ { uniqueCustomers: 20 } ]
   # Ici on verifie que la quantité est supérieur à 0 pour valider une transaction.
   # Dommage que chaque transaction est un client unique...
   ```

7. Répartition des ventes par magasin :

   - Donnez la répartition des ventes pour chaque magasin.

   ```python
   db.sales.aggregate([
     {
       $group: {
         _id: "$store_id",
         totalSales: { $sum: "$total_amount" },
       },
     },
     {
       $sort: { _id: -1 },
     },
   ])

   # ex = [
   #   { _id: 'S003', totalSales: 482.88 },
   #   { _id: 'S002', totalSales: 350.86 },
   #   { _id: 'S001', totalSales: 335.83 }
   # ]

   # On regroupe par magasin puis on additionne ses ventes.
   ```

8. Écart type des montants des transactions :

   - Calculez l'écart type des montants des transactions pour comprendre la variabilité des ventes.

   ```python
   db.sales.aggregate([
     {
       $group: {
         _id: null,
         stdDev: { $stdDevSamp: "$total_amount" }
       }
     },
     {
       $project: {
         _id: 0,
         standardDeviation: "$stdDev"
       }
     }
   ])

   # ex = [{ standardDeviation: 32.914460312020594 }];

   # Sur l'ensemble des documents (_id: null) Mongodb nous fournit l'écart type avec $stdDevSamp!
   ```

9. Distribution des quantités vendues par produit :

   - Analysez la distribution des quantités vendues pour chaque produit. Pensez à faire les calculs de dispersions
     classiques.

   ```python

   ```

10. Médiane des ventes par magasin :

    - Calculez la médiane des montants des transactions pour chaque magasin.

    ```python
    db.sales.aggregate([
      {
        $group: {
          _id: "$store_id",
          amounts: { $push: "$total_amount" },
        },
      },
      {
        $project: {
          _id: 1,
          amounts: { $sortArray: { input: "$amounts", sortBy: 1 } },
        },
      },
      {
        $project: {
          _id: 1,
          medianAmount: {
            $let: {
              vars: {
                size: { $size: "$amounts" },
                middleIndex: { $floor: { $divide: [{ $size: "$amounts" }, 2] } },
              },
              in: {
                $cond: {
                  if: { $eq: [{ $mod: ["$$size", 2] }, 0] }, # Si la taille est paire
                  then: {
                    $avg: [
                      { $arrayElemAt: ["$amounts", "$$middleIndex"] },
                      { $arrayElemAt: ["$amounts", { $subtract: ["$$middleIndex", 1] }] },
                    ],
                  },
                  else: { $arrayElemAt: ["$amounts", "$$middleIndex"] }, # Si la taille est impaire
                },
              },
            },
          },
        },
      },
    ]);

    # ex= [
    #   { _id: 'S001', medianAmount: 47.98 },
    #   { _id: 'S002', medianAmount: 45.98 },
    #   { _id: 'S003', medianAmount: 77.99 }
    # ]

    # $group : Regroupe les montants par magasin. => amounts: [ 33.99, 55.99, 77.99, 133.98, 75.96, 104.97 ]
    # $sortArray : Trie les montants pour chaque magasin. => amounts: [ 33.99, 55.99, 75.96, 77.99, 104.97, 133.98 ]
    # $arrayElemAt : Accède à l'élément médian en calculant la position médiane:
    # - Si la longeur du tableau est impaire on prends la valeur du milieu
    # - SINON on prend les valeurs du milieu et on fait la moyenne
    ```

11. Quartiles des montants des transactions :

    - Déterminez les quartiles pour les montants des transactions afin de mieux comprendre la distribution des ventes.

    ```python

    ```

12. Représenter graphiquement les données
    - Cette question est libre utilisez des graphiques de votre choix pour analyser le dataset.
