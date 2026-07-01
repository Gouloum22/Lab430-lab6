Dyaa Abou Arida

# Rapport Labo 6

**Question 1 : Lequel de ces fichiers Python représente la logique de la machine à états décrite dans les diagrammes du document arc42? Est-ce que son implémentation est complète ou y a-t-il des éléments qui manquent? Illustrez votre réponse avec des extraits de code.**

Le fichier qui représente la logique de la machine à états est src/controllers/order_saga_controller.py, car il garde l’état courant dans self.current_saga_state et fait avancer la saga selon cet état. On voit la transition entre les differents etats.

Code de order_saga_controller:

    while self.current_saga_state is not OrderSagaState.END:
        if self.current_saga_state == OrderSagaState.START:
            self.logger.debug("État initial")
            self.current_saga_state = self.create_order_handler.run()
        elif self.current_saga_state == OrderSagaState.ORDER_CREATED:
            self.decrease_stock_handler = DecreaseStockHandler(self.create_order_handler.order_id, order_data['items'])
            self.current_saga_state = self.decrease_stock_handler.run()
        elif self.current_saga_state == OrderSagaState.STOCK_DECREASED:
            self.create_payment_handler = CreatePaymentHandler(self.create_order_handler.order_id, order_data)
            self.current_saga_state = self.create_payment_handler.run()

![alt text](logs-saga-docker.png)

<p align="center">1.1 Capture d'écran des logs de saga orchestrator</p>

---

**Question 2 : Est-ce que le handler CreateOrderHandler connecte à une base de données directement pour créer des commandes? Illustrez votre réponse avec des extraits de code.**

Non, CreateOrderHandler ne se connecte pas directement à une base de données et il appelle plutôt le service Store Manager à travers une requête. Donc la création réelle de la commande reste dans le microservice store_manager et non pas dans l’orchestrateur.

Code de CreateOrderHandler:

    response = requests.post( f'{config.API_GATEWAY_URL}/store-manager-api/orders', json=self.order_data, headers={'Content-Type': 'application/json'} )

---

**Question 3 : Quelle requête dans la collection Postman du Labo 05 correspond à l'endpoint appelé par CreateOrderHandler? Illustrez votre réponse avec des captures d'écran ou extraits de code.**

La requête Postman du Labo 05 correspondante est la création de commande : POST /store-manager-api/orders. C’est exactement l’endpoint appelé dans CreateOrderHandler.

![alt text](requete-postman-order.png)

<p align="center">3.1 Requête Postman de la creation de commande</p>

---

**Question 4 : Quel endpoint avez-vous appelé pour modifier le stock? Quelles informations de la commande avez-vous utilisées? Illustrez votre réponse avec des extraits de code.**

L’endpoint appelé est le PUT /store-manager-api/stocks via l’API Gateway pour modifier le stock. Les informations utilisées sont les items de la commande, donc chaque product_id avec sa quantity, ainsi que l’opération "-" pour diminuer les quantités.

Code de decrease_stock_handler:

    response = requests.put(
        f'{config.API_GATEWAY_URL}/store-manager-api/stocks',
        json={
            "items": self.order_item_data,
            "operation": "-"
        },
        headers={'Content-Type': 'application/json'}
    )

---

**Question 5 : Quel endpoint avez-vous appelé pour générer une transaction de paiement? Quelles informations de la commande avez-vous utilisées? Illustrez votre réponse avec des extraits de code.**

L'endpoint appelé est le POST /payments-api/payments via l’API Gateway pour créer la transaction de paiement. Les informations utilisées sont donc le user_id, le order_id et le montant total de la commande.

Code de create_payment_handler:

    payment_response = requests.post(
        f'{config.API_GATEWAY_URL}/payments-api/payments',
        json={
            "user_id": self.order_data.get("user_id"),
            "order_id": self.order_id,
            "total_amount": self.total_amount
        },
        headers={'Content-Type': 'application/json'}
    )

---

**Question 6 : Quelle est la différence entre appeler l'orchestrateur Saga et appeler directement les endpoints des services individuels? Quels sont les avantages et inconvénients de chaque approche? Illustrez votre réponse avec des captures d'écran ou extraits de code.**

L’orchestrateur Saga sert à appeler un seul endpoint, /saga/order ce qui gère automatiquement la création de commande, la diminution du stock et la création du paiement. Pour les appels de services individuels, le client doit gérer l’ordre des appels et les erreurs. L’avantage de la Saga est une meilleure logique, mais l’inconvénient est une complexité plus grande. Les appels directs sont plus simples, mais plus risqués si un service échoue au milieu du processus.

![alt text](image.png)

<p align="center">6.1 Trace de Jaeger UI</p>

![alt text](image-1.png)

<p align="center">6.2 Trace de tous les services</p>

![alt text](image-2.png)

<p align="center">6.3 Gestion des échecs</p>
