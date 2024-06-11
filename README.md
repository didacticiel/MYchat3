MYchat3

Application de chat en temps réel utilisant Django
Dernière mise à jour : 27 mars 2024

La création d'une salle de chat est l'étape la plus basique pour la création de projets en temps réel et en direct. La page de chat que nous allons créer sera une simple structure HTML avec un titre h1 indiquant le nom de l'utilisateur actuel et un lien de déconnexion pour l'utilisateur connecté. Vous devrez peut-être commenter la ligne jusqu'à ce que nous créions un système d'authentification pour cela. Cela garantit que lorsque deux utilisateurs discutent, l'un peut se déconnecter sans affecter l'autre utilisateur. L'autre sera toujours connecté et pourra envoyer et recevoir des messages.


Prérequis :
Django
Migrations Django
Canal Django


Introduction aux canaux Django et à la programmation asynchrone

Les canaux sont le projet Python créé pour étendre la capacité de Django au niveau supérieur. Nous travaillions avec Django standard qui ne supportait pas les connexions asynchrones et via WebSockets pour créer des applications en temps réel. Les canaux étendent la capacité de Django au-delà de HTTP et le font fonctionner avec WebSockets, les protocoles de chat, les protocoles IoT, et plus encore. Il est construit sur le support ASGI qui signifie Asynchronous Server Gateway Interface. ASGI est le successeur de WSGI qui fournit une interface entre async et Python. Les canaux offrent la fonctionnalité d'ASGI en étendant WSGI, et ils offrent un support ASGI avec WSGI. Les canaux regroupent également l'architecture événementielle avec les couches de canaux, un système qui permet de communiquer facilement entre les processus et de séparer votre projet en différents processus.



Étapes pour créer l'application de chat
Étape 1 : Installer et configurer Django
Étape 2 : Créer votre environnement virtuel.
Étape 3 : Créer un projet Django nommé ChatApp. Pour créer le projet, écrivez la commande suivante dans votre terminal :




django-admin startproject ChatApp
Étape 4 : Après la création du projet, la structure du dossier devrait ressembler à ceci, puis ouvrez votre IDE préféré pour continuer.
Étape 5 : Installer django-channels pour travailler avec l'application de chat. Cela installera les canaux dans votre environnement.




python -m pip install -U channels
Note : À partir de la version 4.0.0 des canaux, ASGI runserver en mode développement ne fonctionne plus. Vous devrez également installer daphne.





python -m pip install -U daphne
Après avoir installé daphne, ajoutez-le à vos applications installées.




INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # ajouter django daphne
    'daphne',
]


Étape 6 : Après avoir installé les canaux, ajoutez les canaux à vos applications installées. Cela permettra à Django de savoir que les canaux ont été introduits dans le projet et nous pourrons continuer.





INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # ajouter django channels
    'channels',
]



Étape 7 : Définir l'application ASGI sur votre fichier ASGI par défaut dans le projet. Maintenant, exécutez le serveur, vous remarquerez que le serveur ASGI prendra la place du serveur Django et il prendra désormais en charge ASGI. Enfin, définissez votre paramètre ASGI_APPLICATION pour pointer vers cet objet de routage en tant qu'application racine :
python






from channels.routing import ProtocolTypeRouter
from django.core.asgi import get_asgi_application

django_asgi_app = get_asgi_application()

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "ChatApp.settings")
application = ProtocolTypeRouter({
    "http": django_asgi_app,
    # Juste HTTP pour l'instant. (Nous pouvons ajouter d'autres protocoles plus tard.)
})


ASGI_APPLICATION = 'ChatApp.asgi.application'
Pour exécuter le serveur, écrivez la commande suivante dans le terminal :




python3 manage.py runserver
Étape 8 : Créer une nouvelle application qui aura toute la fonctionnalité de chat. Pour créer une application, écrivez une commande dans le terminal :





python manage.py startapp chat
Ajoutez votre application aux applications installées dans settings.py.

Étape 9 : Créer quelques fichiers dans votre application de chat
chat/urls.py : Cela dirigera l'application Django vers différentes vues dans l'application.
Créer un dossier templates : À l'intérieur de votre application, créez deux fichiers dans le dossier template/chat nommés chatPage.html et LoginPage.html.
routing.py : Cela dirigera les connexions WebSocket vers les consommateurs.
consumers.py : C'est le fichier où toute la fonctionnalité asynchrone aura lieu.
Collez ce code dans votre fichier ChatApp/urls.py. Cela vous dirigera vers votre application de chat.




from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path("", include("chat.urls")),
]
Collez ce code dans votre fichier chat/urls.py. Cela vous dirigera vers les vues.





from django.urls import path, include
from chat import views as chat_views
from django.contrib.auth.views import LoginView, LogoutView

urlpatterns = [
    path("", chat_views.chatPage, name="chat-page"),

    # section de connexion
    path("auth/login/", LoginView.as_view
         (template_name="chat/LoginPage.html"), name="login-user"),
    path("auth/logout/", LogoutView.as_view(), name="logout-user"),
]
Collez ce code dans votre fichier views.py. Cela dirigera vos vues vers chatPage.html qui a été créé dans le dossier templates de l'application de chat.





from django.shortcuts import render, redirect

def chatPage(request, *args, **kwargs):
    if not request.user.is_authenticated:
        return redirect("login-user")
    context = {}
    return render(request, "chat/chatPage.html", context)
L'URL est au format Django, c'est la syntaxe Django pour mapper une URL. Nous créerons une URL nommée “logout-user”, puis Django mappera ce nom d'URL à l'URL du modèle. Django fournit quelques syntaxes pythonic pour gérer l'instruction de contrôle. Ici, nous avons fourni la ligne {% if request.user.is_authenticated %} dans le HTML, cela est donné par Django qui garantit que s'il y a un utilisateur connecté, il affiche uniquement le lien de déconnexion.


templates/chat/chatPage.html




<!DOCTYPE html>
<html>
  <body>
    <center><h1>Bonjour, Bienvenue sur mon site de chat ! {{request.user}}</h1></center>
    <br>
    {% if request.user.is_authenticated  %}
    <center> Déconnectez-vous de la page de chat <a href = "{% url 'logout-user' %}">Logout</a></center>
    {% endif %}
    <div
      class="chat__item__container"
      id="id_chat_item_container"
      style="font-size: 20px"
    >
      <br />
      <input type="text" id="id_message_send_input" />
      <button type="submit" id="id_message_send_button">Send Message</button>
      <br />
      <br />
    </div>
    <script>
      const chatSocket = new WebSocket("ws://" + window.location.host + "/");
      chatSocket.onopen = function (e) {
        console.log("La connexion a été établie avec succès !");
      };
      chatSocket.onclose = function (e) {
        console.log("Quelque chose d'inattendu s'est produit !");
      };
      document.querySelector("#id_message_send_input").focus();
      document.querySelector("#id_message_send_input").onkeyup = function (e) {
        if (e.keyCode == 13) {
          document.querySelector("#id_message_send_button").click();
        }
      };
      document.querySelector("#id_message_send_button").onclick = function (e) {
        var messageInput = document.querySelector(
          "#id_message_send_input"
        ).value;
        chatSocket.send(JSON.stringify({ message: messageInput, username : "{{request.user.username}}"}));
      };
      chatSocket.onmessage = function (e) {
        const data = JSON.parse(e.data);
        var div = document.createElement("div");
        div.innerHTML = data.username + " : " + data.message;
        document.querySelector("#id_message_send_input").value = "";
        document.querySelector("#id_chat_item_container").appendChild(div);
      };
    </script>
  </body>
</html>
{{request.user.userrname}} affiche le nom d'utilisateur de l'utilisateur actuellement connecté. Si l'utilisateur est connecté, il affichera son nom d'utilisateur ; s'il n'est pas connecté, il n'affichera rien. La page de chat ressemble à ceci maintenant, car il n'y a pas d'utilisateur actuellement connecté et {{request.user.username}} n'affiche rien.



templates/chat/LoginPage.html



<!DOCTYPE html>
<html>
<body>
    <form method ="post">
        {% csrf_token %}
        {{form.as_p}}
        <br>
        <button type = "submit">Login</button>
    </form>
</body>
</html>
Implémenter les WebSockets, asynchrone et les canaux Django
Jusqu'à présent, nous avons configuré notre projet Django standard. Après avoir implémenté l'application Django, il est temps de créer l'application ASGI.


Étape 10 : Premièrement, migrez votre base de données.



python manage.py makemigrations
python manage.py migrate
Étape 11 : Ouvrez routing.py et créez une route pour ChatConsumer (que nous allons créer à l'étape suivante). Nous avons maintenant deux types de routage dans le projet. Le premier est urls.py qui est pour le routage natif de Django des URLs, et l'autre est pour les WebSockets pour le support ASGI de Django.



from django.urls import path , include
from chat.consumers import ChatConsumer

# Ici, "" est routé vers l'URL ChatConsumer qui
# gérera la fonctionnalité de chat.
websocket_urlpatterns = [
    path("" , ChatConsumer.as_asgi()) , 
] 
Étape 12 : Ouvrez consumers.py pour gérer les événements, comme l'événement onmessage, l'événement onopen, etc. Nous verrons ces événements dans chatPage.html où nous avons créé la connexion socket.
Explication du code :




class ChatConsumer(AsyncWebsocketConsumer): Ici, nous créons une classe nommée ChatConsumer qui hérite d'AsyncWebsocketConsumer et est utilisée pour créer, détruire et faire quelques autres choses avec les WebSockets. Et ici, nous créons ChatSocket pour l'objectif requis.
async def connect(self): Cette fonction fonctionne sur l'instance de websocket qui a été créée et lorsque la connexion est ouverte ou créée, elle se connecte et accepte la connexion. Elle crée un nom de groupe pour la salle de chat et ajoute le groupe à la couche de canal.
async def disconnect(): Cela supprime simplement l'instance du groupe.
async def receive(): Cette fonction est déclenchée lorsque nous envoyons des données depuis le WebSocket (l'événement pour que cela fonctionne est : envoyer), cela reçoit les données texte qui ont été converties en format JSON (car il est adapté pour le JavaScript) après que le texte_data a été reçu, puis il doit être diffusé aux autres instances qui sont actives dans le groupe. nous récupérons le paramètre message qui contient le message et le paramètre username qui a été envoyé par le socket via HTML ou js. Ce message reçu sera diffusé aux autres instances via la méthode channel_layer.group_send() qui prend comme premier argument le roomGroupName auquel appartient cette instance et où les données doivent être envoyées. puis le deuxième argument est le dictionnaire qui définit la fonction qui gérera l'envoi des données ( "type": "sendMessage" ) et également le dictionnaire a la variable message qui contient les données du message.
async def sendMessage(self, event): Cette fonction prend l'instance qui envoie les données et l'événement, essentiellement l'événement contient les données qui ont été envoyées via la méthode group_send() de la fonction receive(). Ensuite, elle envoie le message et le paramètre username à toutes les instances qui sont actives dans le groupe. Et il est converti en format JSON pour que js puisse comprendre la notation. JSON est le format (JavaScript Object Notation)
chat/consumers.py



import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.roomGroupName = "group_chat_gfg"
        await self.channel_layer.group_add(
            self.roomGroupName,
            self.channel_name
        )
        await self.accept()
    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.roomGroupName,
            self.channel_layer
        )
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]
        username = text_data_json["username"]
        await self.channel_layer.group_send(
            self.roomGroupName,{
                "type" : "sendMessage",
                "message" : message,
                "username" : username,
            })
    async def sendMessage(self, event): 
        message = event["message"]
        username = event["username"]
        await self.send(text_data = json.dumps({"message":message, "username":username}))
Le ChatConsumer que nous mappons dans routing.py est le même que celui que nous avons créé dans consumers.py. Ces scripts sont en mode asynchrone, c'est pourquoi nous travaillons avec async et await ici.

Étape 13 : Écrivez le code ci-dessous dans votre asgi.py pour le faire fonctionner avec les sockets et créer les routages.
ChatApp/asgi.py



import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'ChatApp.settings')

from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from chat import routing

application = ProtocolTypeRouter(
    {
        "http" : get_asgi_application(), 
        "websocket" : AuthMiddlewareStack(
            URLRouter(
                routing.websocket_urlpatterns
            )    
        )
    }
)
Nous travaillons habituellement avec wsgi.py qui est dans le Django standard sans support asynchrone. Mais ici nous utilisons des canaux asynchrones. Nous devons donc définir les routages d'une manière différente de celle des URL. Pour HTTP, nous définissons l'application normale que nous utilisions déjà, maintenant nous avons introduit un autre protocole, qui est ws (WebSocket) pour lequel vous devez router. Le ProtocolTypeRouter crée des routes pour différents types de protocoles utilisés dans l'application. AuthMiddlewareStack authentifie les routes et les instances pour l'authentification et URLRouter route les ws (connexions WebSocket). Le protocole pour les WebSockets est connu sous le nom de "ws". Pour différentes requêtes, nous utilisons HTTP.

Ici, le routeur dirige l'URL WebSocket vers une variable dans l'application de chat qui est "websocket_urlpatterns" et cette variable contient les routes pour les connexions WebSocket.

Étape 14 : Ce code définit la couche de canal dans laquelle nous allons travailler et partager des données. Pour le déploiement et le niveau de production, n'utilisez pas InMemoryChannelLayer, car il y a de grandes chances de fuite de données. Cela n'est pas bon pour la production. Pour la production, utilisez la couche de canal Redis.
ChatApp/settings.py



CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer"
    }
}


Étape 15 : Maintenant, nous devons créer 2 utilisateurs pour cela nous utiliserons la commande “python manage.py createsuperuser” qui crée un superutilisateur dans le système.
Étape 16 : Nous avons défini le paramètre LOGIN_REDIRECT_URL = “chat-page”, qui est le nom de notre URL de page d'atterrissage. Cela signifie que chaque fois que l'utilisateur se connecte, il sera redirigé vers la page de chat en tant qu'utilisateur vérifié et il est éligible pour discuter. De même, nous devons configurer l'URL de redirection de déconnexion pour le site.
ChatApp/settings.py

Déploiement
Maintenant, exécutez votre serveur, allez sur le site et commencez avec deux navigateurs différents pour vous connecter avec deux utilisateurs différents. Cela est dû au fait que si vous vous connectez avec les identifiants du premier utilisateur, les détails de connexion sont stockés dans les cookies, puis si vous vous connectez avec les identifiants du second utilisateur dans le même navigateur même avec des onglets différents, vous ne pouvez pas discuter avec deux autres utilisateurs dans le même navigateur, c'est pourquoi utilisez deux navigateurs différents.