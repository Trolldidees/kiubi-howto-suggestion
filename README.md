# Suggestions de produits 

## Introduction

Ce dépôt est un tutoriel qui explique comment utiliser l'API de son site [Kiubi](http://www.kiubi.com) pour afficher une liste de produits comme suggestion d'achat pour l'internaute.

Nous allons lui proposer des produits en relation avec sa dernière commande lorsqu'il se connecte, puis des produits associés à ceux présents dans son panier, et enfin nous allons créer un bloc contenant les derniers produits visités.

## Prérequis

Ce tutoriel suppose que vous avez un site Kiubi et qu'il est bien configuré : 

 - l'API est activée
 - le catalogue et l'ecommerce activés 
 - des produits configurés avec des produits associés
 - le widget panier est inséré dans l'entête des mises en page
 - le site est en thème personnalisé, basé sur Shiroi

Il est préférable d'être à l'aise avec la manipulation des thèmes personnalisés. En cas de besoin, le [guide du designer](http://doc.kiubi.com) est là.

Ce tutoriel est applicable à tout thème graphique mais les exemples de codes donnés sont optimisés pour un rendu basé sur le thème Shiroi.


## Inclusions des bibliothèques javascript

Pour réaliser notre exemple il nous faut 2 librairies, le client JS Kiubi pour utiliser l’API et un plugin jQuery qui nous permettra de gérer facilement un cookie pour la mémorisation des produits visités.

Le plugin jQuery utilisé est jquery-cookie et est distribué sous licence MIT. Les sources sont disponibles à l’adresse suivante : [https://github.com/carhartl/jquery-cookie/](https://github.com/carhartl/jquery-cookie) 
On rajoute l’inclusion des deux fichiers grâce au module `Injection de code` et on met le code d’inclusion avant la balise `</head>` : 

<pre lang="html">
&lt;script type="text/javascript" src="//{cdn}/js/kiubi.api.pfo.jquery-1.0.min.js">&lt;/script>
&lt;script type="text/javascript" src="//cdnjs.cloudflare.com/ajax/libs/jquery-cookie/1.3.1/jquery.cookie.min.js">&lt;/script>
</pre>

*Nous utilisons ici un cdn public tiers pour l’inclusion du plugin jQuery, vous pouvez également récupérer le fichier et le déposer dans votre thème puis l’inclure dans vos templates de pages.*

## Suggestions de produits à la connexion

Nous allons modifier le template du panier afin de pouvoir se conncter sans rechargement de la page. Une fois connecté, nous afficherons le statut de la dernière commande de l'intenaute et nous lui proposerons des produits en relation avec ceux déjà achetés.

### Mise en place 

Remplacez le template `theme/fr/widgets/commandes/panier/index.html` de votre thème par celui de ce tutoriel.

Allez sur la page d'accueil et cliquez sur le lien `Se connecter` en haut à droite. Le mini formulaire de connexion doit s'afficher. Si vous vous connectez et que vous avez déjà passé une commande, une liste de produit associés doit s'afficher sous l'entête. 

### Explications

Examinons en détail le code HTML du template `panier/index.html` :

* Dans le bloc `main.nonidentifie` on ajoute l'html suivant pour faire un mini formulaire de connexion qui est par défaut masqué :

<pre lang="html">
&lt;span style="display:none;"> : 
  &lt;label for="api-login">Identifiant : &lt;/label>&lt;input type="text" class="field" id="api-login" size="10"/> 
  &lt;label for="api-pwd">Mot de passe : &lt;/label>&lt;input type="password" class="field" id="api-pwd" size="10"/>
  &lt;input id="api-submit" type="button" class="submit" style="padding:2px;" value="Ok" />
&lt;/span>
</pre>

On se greffe sur l'événement `click` du lien `Se connecter` pour changer l'état d'affichage de notre mini formulaire :

<pre lang="javascript">
$(document).on('click', '#api-link-login', function(){
	$('p.login > span').toggle();
	return false;
});
</pre>

Lorsque l'on valide le mini formulaire, on effectue une connexion à Kiubi via l'API, si celle ci échoue, on change le style des champs du formulaire. Si l'utilisateur est connecté on affiche son nom, un lien pour se déconnexcter puis on récupère ses dernières commandes via la méthode `getOrders()`

<pre lang="javascript">
$(document).on('click', "#api-submit", function(){
  var login = $("#api-login").val();
  var pwd = $("#api-pwd").val();
  
  // Pas de requête si aucun login ou aucun mot de passe
  if(login == '' || pwd == '') {
    $("p.login input.field").css('border', '2px solid red');
    return;
  }
  
  // Connexion à kiubi
  var query = kiubi.login(login, pwd);
  
  // En cas de connexion réussie
  query.done(function(meta, data){
    $(".login").hide().after('&lt;p class="compte">&lt;a href="{baseLangue}/compte/">'+data.firstname+' '+data.lastname+'&lt;/a> - &lt;a id="api-link-logout" href="{baseLangue}/compte/logout.html">D&eacute;connexion&lt;/a>&lt;/p>');
    
    getOrders(data.user_id);
  });
  
  // En cas de connexion échouée
  query.fail(function(meta, error){
    $("p.login input.field").css('border', '2px solid red');
  });         
});
</pre>

Si l'utilisateur clique sur le lien de déconnexion on éffectue celle-ci via l'API :

<pre lang="javascript">
$(document).on('click', '#api-link-logout', function(){
	// Déconnexion de l'utilisateur courant
	kiubi.logout().done(function(meta, data){
		$("p.compte").hide();
		$("p.login").show();
		$("p.login span").hide();
		$("p.login input.field").removeAttr('style');
		$("#api-pwd").val('');
	});
	return false;
});
</pre>

Dans le bloc `main.identifie` on se contente de récupérer les dernières commandes du membre :

<pre lang="javascript">
getOrders({CLIENT.client_id});
</pre>

La fonction `getOrders()` va réaliser plusieurs opérations :
 
- récupération des commandes et affichage du statut de la dernière
- récupération du détail de la dernière commande
- récupération des produits associés aux produits de cette commande
- affichage de ces produits avec la possibilité de les ajouter au panier en un clic
- permettre à l'utilisateur de masquer cet encart de suggestion

On récupère les commandes uniquement si l'utilisateur n'a pas demandé la fermeture de l'encart de suggestions. Une fois la liste récupérée, on affiche devant son nom le statut de sa dernière commande s'il en a une.

<pre lang="javascript">
if($.cookie('nosuggestion')=='1') return;
  kiubi.users.getOrders(id).done(function(meta, data){
    if(data.length) {
      $('p.compte > a:first').before('Etat de votre dernière commande : '+data[0].status+' - ');
</pre> 

Ensuite on récupère le détail de la dernière commande, et on stocke les identifiants de chaque produit acheté :

<pre lang="javascript">
kiubi.users.getOrder(data[0].id).done(function(meta, data){
	var products = [];
	for(var i=0; i&lt;data.items.length; i++) {
		if(data.items[i].product_id)
			products.push(data.items[i].product_id);
	}
</pre>

Via l'API on requête la liste des produits associés de chacun des produits achetés, on stocke chaque produit associé qui n'a pas été acheté dans un tableau, dans la limite de 3 pour des questions d'affichage dans notre thème :

<pre lang="javascript">
var linked = [], queries = [];
for(var i=0; i&lt;products.length &amp;&amp; linked.length&lt;3; i++) {
  queries.push(kiubi.catalog.getLinkedProducts(products[i], {extra_fields:'price_label,variants'}).done(function(meta, data){
    for(var p=0; p&lt;data.length; p++) {
      if(linked.length&lt;3 &amp;&amp; $.inArray(data[p].id, products)==-1) {
        linked.push(data[p]);
      }
    }             
  }));            
}
</pre>

Une fois que toutes les requêtes vers tous les produits ont été effectués, on construit l'html de notre liste de suggestions avec les données stockées précédemment, puis on les affiche en dessous de notre entête : 

<pre lang="javascript">
$.when.apply($, queries).done(function(){           
  if(linked.length) {
    var html = '&lt;div style="display:none;" id="suggestions">&lt;br/>&lt;span style="float:right" id="closesuggestion">X&lt;/span>&lt;div>&lt;strong>Voici une liste de produits pour compléter votre dernière commande :&lt;/strong>&lt;br/>';
    for(var p=0; p&lt;linked.length; p++) {
      html += '&lt;div style="float:left;width:'+(100/linked.length)+'%">';
      html += '&lt;h4>&lt;a href="'+linked[p].url+'">'+linked[p].name+'&lt;/a> - '+linked[p].price_min_inc_vat_label+'&lt;/h4>';
      html += '&lt;a style="float:left;margin-right:10px;" href="'+linked[p].url+'">&lt;img src="'+linked[p].main_thumb.url_miniature+'"/>&lt;/a>';
      html += '&lt;div>'+linked[p].header+'&lt;/div>';
      html += '&lt;input class="bouton addtocart" type="submit" data-id="'+linked[p].variants[0].id+'" value="Ajouter au panier"/>';
      html += '&lt;/div>';
    }
    html += '&lt;/div>&lt;/div>';
    $('.menu_header').append(html);
    $('#suggestions').slideDown('slow');
</pre>

On se greffe sur l'événement `click` des boutons ajoutés dans notre liste de suggestion pour ajouter directement le produit dans le panier de l'internaute :

<pre lang="javascript">
$('.addtocart').click(function(){
  var me = $(this);
  me.next('p').remove();
  kiubi.cart.addItem($(this).data('id'), '+1', null, {extra_fields:'price_label'}).done(function(meta, data){
    me.after('&lt;p>Produit ajouté au panier !&lt;/p>');
    $('.panier_link a').html(data.cart.items_count+' article'+(data.cart.items_count>1?'s':'')+' - '+data.cart.price_total_inc_vat_label);
  });
});
</pre>

Enfin, sur le clic de la croix ajoutée à droite de notre liste on masque les suggestions et on stocke un cookie afin de ne pas les représenter au visiteur :

<pre lang="javascript">
$("#closesuggestion").click(function(){
  $('#suggestions').slideUp('fast');
  $.cookie('nosuggestion', '1');
});
</pre>

### Exemple complet

<pre lang="html">
&lt;!-- 
Template du Widget "Panier"
-->

&lt;!-- BEGIN:main -->
&lt;div class="panier">
  &lt;!-- BEGIN:nonidentifie -->
  &lt;p class="login">&lt;a id="api-link-login" href="{baseLangue}/compte/">Se connecter&lt;/a>
    &lt;span style="display:none;"> : 
      &lt;label for="api-login">Identifiant : &lt;/label>&lt;input type="text" class="field" id="api-login" size="10"/> 
      &lt;label for="api-pwd">Mot de passe : &lt;/label>&lt;input type="password" class="field" id="api-pwd" size="10"/>
      &lt;input id="api-submit" type="button" class="submit" style="padding:2px;" value="Ok" />
    &lt;/span>
  &lt;/p>
  &lt;script type="text/javascript">
    $(function(){
      // Interception du clic sur le lien "Se connecter" 
      // et affichage du mini formulaire de connexion
      $(document).on('click', '#api-link-login', function(){
        $('p.login > span').toggle();
        return false;
      });
      
      // Login sur le click du bouton du mini formulaire
      $(document).on('click', "#api-submit", function(){
        var login = $("#api-login").val();
        var pwd = $("#api-pwd").val();
        
        // Pas de requête si aucun login ou aucun mot de passe
        if(login == '' || pwd == '') {
          $("p.login input.field").css('border', '2px solid red');
          return;
        }
        
        // Connexion à kiubi
        var query = kiubi.login(login, pwd);
        
        // En cas de connexion réussie
        query.done(function(meta, data){
          $(".login").hide().after('&lt;p class="compte">&lt;a href="{baseLangue}/compte/">'+data.firstname+' '+data.lastname+'&lt;/a> - &lt;a id="api-link-logout" href="{baseLangue}/compte/logout.html">D&eacute;connexion&lt;/a>&lt;/p>');
          
          getOrders(data.user_id);
        });
        
        // En cas de connexion échouée
        query.fail(function(meta, error){
          $("p.login input.field").css('border', '2px solid red');
        });         
      });
      
      // interception du clic sur le lien "Déconnexion"
      $(document).on('click', '#api-link-logout', function(){
        
        // Déconnexion de l'utilisateur courant
        kiubi.logout().done(function(meta, data){
          $("p.compte").hide();
          $("p.login").show();
          $("p.login span").hide();
          $("p.login input.field").removeAttr('style');
          $("#api-pwd").val('');
        });
        return false;
      });
    });
  &lt;/script>
  &lt;!-- END:nonidentifie -->
  &lt;!-- BEGIN:identifie -->
  &lt;p class="compte">&lt;a href="{baseLangue}/compte/">{prenom} {nom}&lt;/a> - &lt;a href="{baseLangue}/compte/logout.html">D&eacute;connexion&lt;/a>&lt;/p>
  &lt;script type="text/javascript">
    $(function(){
      getOrders({CLIENT.client_id});
    });
  &lt;/script>
  &lt;!-- END:identifie -->
  &lt;p class="panier_link">&lt;a href="{baseLangue}/ecommerce/panier.html" title="Détail du panier">{nb_produits} article{pluriel_produits} - {total}&lt;/a>&lt;/p>
&lt;/div>
&lt;script type="text/javascript">
getOrders = function(id) {
  if($.cookie('nosuggestion')=='1') return;
  kiubi.users.getOrders(id).done(function(meta, data){
    if(data.length) {
      $('p.compte > a:first').before('Etat de votre dernière commande : '+data[0].status+' - ');

      kiubi.users.getOrder(data[0].id).done(function(meta, data){
        var products = [];
        for(var i=0; i&lt;data.items.length; i++) {
          if(data.items[i].product_id)
            products.push(data.items[i].product_id);
        }
        if(products.length) {
          var linked = [], queries = [];
          for(var i=0; i&lt;products.length &amp;&amp; linked.length&lt;3; i++) {
            queries.push(kiubi.catalog.getLinkedProducts(products[i], {extra_fields:'price_label,variants'}).done(function(meta, data){
              for(var p=0; p&lt;data.length; p++) {
                if(linked.length&lt;3 &amp;&amp; $.inArray(data[p].id, products)==-1) {
                  linked.push(data[p]);
                }
              }             
            }));            
          }         
          $.when.apply($, queries).done(function(){           
            if(linked.length) {
              var html = '&lt;div style="display:none;" id="suggestions">&lt;br/>&lt;span style="float:right" id="closesuggestion">X&lt;/span>&lt;div>&lt;strong>Voici une liste de produits pour compléter votre dernière commande :&lt;/strong>&lt;br/>';
              for(var p=0; p&lt;linked.length; p++) {
                html += '&lt;div style="float:left;width:'+(100/linked.length)+'%">';
                html += '&lt;h4>&lt;a href="'+linked[p].url+'">'+linked[p].name+'&lt;/a> - '+linked[p].price_min_inc_vat_label+'&lt;/h4>';
                html += '&lt;a style="float:left;margin-right:10px;" href="'+linked[p].url+'">&lt;img src="'+linked[p].main_thumb.url_miniature+'"/>&lt;/a>';
                html += '&lt;div>'+linked[p].header+'&lt;/div>';
                html += '&lt;input class="bouton addtocart" type="submit" data-id="'+linked[p].variants[0].id+'" value="Ajouter au panier"/>';
                html += '&lt;/div>';
              }
              html += '&lt;/div>&lt;/div>';
              $('.menu_header').append(html);
              $('#suggestions').slideDown('slow');
              $('.addtocart').click(function(){
                var me = $(this);
                me.next('p').remove();
                kiubi.cart.addItem($(this).data('id'), '+1', null, {extra_fields:'price_label'}).done(function(meta, data){
                  me.after('&lt;p>Produit ajouté au panier !&lt;/p>');
                  $('.panier_link a').html(data.cart.items_count+' article'+(data.cart.items_count>1?'s':'')+' - '+data.cart.price_total_inc_vat_label);
                });
              });
              $("#closesuggestion").click(function(){
                $('#suggestions').slideUp('fast');
                $.cookie('nosuggestion', '1');
              });
            }
          });         
        }
      });
    }           
  });
} 
&lt;/script>
&lt;!-- END:main -->
</pre>

## Suggestions de produits dans le panier

Nous allons modifier le template du détail panier afin d'insérer un produit associé à chaque produit présent dans le panier, en veillant à ne pas proposer un produit déjà présent dans le panier.

### Mise en place

Remplacez le template `theme/fr/widgets/commandes/panier_detail/index.html` de votre thème par celui de ce tutoriel.

Mettre un produit dans votre panier. La page de détail du panier doit afficher 2 lignes, votre produit ajouté et une suggestion.

### Explications

Examinons en détail le code HTML du template `panier_detail/index.html` :

On commence par récupérer le panier via l'API afin de stocker l'identifiant de chaque produit du panier :

<pre lang="javascript">
kiubi.cart.get().done(function(meta, data){
  pid = [];
  for(var i=0; i&lt;data.items.length; i++) {
    pid.push(data.items[i].product_id);
  }
</pre>

Ensuite, pour chacun des produits on va récupérer ses produits associés, et stocker dans l'objet `promise` de notre requête l'identifiant de la variante d'origine sous la variable `vid` ainsi que le nom du produit afin de pouvoir les utiliser plus tard :

<pre lang="javascript">
for(var i=0; i&lt;data.items.length; i++) {
  var q = kiubi.catalog.getLinkedProducts(data.items[i].product_id, {random_fill:false,extra_fields:'price_label,variants'});
  q.promise.vid = data.items[i].variant_id;       
  q.promise.pname = data.items[i].product_name;     
</pre>

Lorsque la requête est exécutée, la méthode `done()` est appelée. Si un produit associé, non présent dans le panier, a été trouvé et qu'il est disponible et en stock, on crée une nouvelle ligne de panier avec le produit associé et un bouton pour l'ajouter au panier. On insère ensuite cette ligne après la ligne du produit auquel elle se rapporte grâce à l'identifiaction de la variante que nous avions stocké dans notre objet `promise` et que nous accédons avec le code `this.promise.vid` :

<pre lang="javascript">
q.done(function(meta, data){      
  if(!data.length) return;
  for(var p=0; p&lt;data.length; p++) {  
    if($.inArray(data[p].id, pid)==-1 &amp;&amp; data[p].variants[0].is_available &amp;&amp; data[p].variants[0].in_stock) {
      pid.push(data[p].id);
      var html = '&lt;tr>&lt;td class="produit_illustration">&lt;a href="'+data[p].url+'">&lt;img src="'+data[p].main_thumb.url_miniature+'"/>&lt;/a>&lt;/td>';
      html += '&lt;td class="produit_info">&lt;strong>Nous vous proposons en complément de '+this.promise.pname+' :&lt;/strong>&lt;br/>&lt;strong>&lt;a href="'+data[p].url+'">'+data[p].name+"&lt;/strong>&lt;/a>&lt;br/>&lt;p>"+data[p].variants[0].name+'&lt;/p>&lt;p class="desc">'+data[p].header+'&lt;/p>&lt;/td>';
      html += '&lt;td>'+data[p].variants[0].price_inc_vat_label+'&lt;/td>';
      html += '&lt;td nowrap="nowrap" class="produit_qte">&lt;input type="text" class="textfield" value="1" style="width:30px" id="qt-'+data[p].variants[0].id+'"/>&lt;/td>';                  
      html += '&lt;td colspan="2">&lt;input type="submit" style="margin-top:0px;" class="addsuggest" data-id="'+data[p].variants[0].id+'" value="Ajouter au panier"/>';
      html += '&lt;/td>&lt;/tr>';
      $('#p-'+this.promise.vid).after(html);
      break;                
    }
  }
});
</pre>

On se greffe sur l'événement `click` des boutons que nous venons d'insérer afin d'ajouter les produits au panier via l'API et recharger la page afin d'actualiser l'affichage du panier :

<pre lang="javascript">
$(document).on('click', '.addsuggest', function(){
	kiubi.cart.addItem($(this).data('id'), $('#qt-'+$(this).data('id')).val()).done(function(){document.location.reload();});
});
</pre>

La seule modification html apportée au template d'origine et l'ajout d'un id sur chaque ligne produit du panier afin de pouvoir repérer la variante et insérer à la suite sa suggestion :

<pre lang="html">
&lt;!-- BEGIN:produit -->
&lt;tr id="{ref}">
</pre>


### Exemple complet

<pre lang="html">
&lt;!-- 
Template du Widget "Détail du panier"
-->
&lt;!-- BEGIN:main -->

&lt;ul class="chemin_commande">
  &lt;li>&lt;a href="{baseLangue}/ecommerce/panier.html" title="Panier" class="actif">1. Panier&lt;/a>&lt;/li>
  &lt;li>2. Authentification&lt;/li>
  &lt;li>3. Livraison&lt;/li>
  &lt;li>4. Mode de paiement&lt;/li>
  &lt;li>5. Validation&lt;/li>
&lt;/ul>
&lt;article class="panier_detail">
  &lt;!-- BEGIN:intitule -->
  &lt;h1>{intitule}&lt;/h1>
  &lt;!-- END:intitule -->
  &lt;!-- BEGIN:produits -->
  &lt;script type="text/javascript">
  $(function(){
    kiubi.cart.get().done(function(meta, data){
      pid = [];
      for(var i=0; i&lt;data.items.length; i++) {
        pid.push(data.items[i].product_id);
      }
      for(var i=0; i&lt;data.items.length; i++) {
        var q = kiubi.catalog.getLinkedProducts(data.items[i].product_id, {random_fill:false,extra_fields:'price_label,variants'});
        q.promise.vid = data.items[i].variant_id;       
        q.promise.pname = data.items[i].product_name;       
        q.done(function(meta, data){      
          if(!data.length) return;
          for(var p=0; p&lt;data.length; p++) {  
            if($.inArray(data[p].id, pid)==-1 &amp;&amp; data[p].variants[0].is_available &amp;&amp; data[p].variants[0].in_stock) {
              pid.push(data[p].id);
              var html = '&lt;tr>&lt;td class="produit_illustration">&lt;a href="'+data[p].url+'">&lt;img src="'+data[p].main_thumb.url_miniature+'"/>&lt;/a>&lt;/td>';
              html += '&lt;td class="produit_info">&lt;strong>Nous vous proposons en complément de '+this.promise.pname+' :&lt;/strong>&lt;br/>&lt;strong>&lt;a href="'+data[p].url+'">'+data[p].name+"&lt;/strong>&lt;/a>&lt;br/>&lt;p>"+data[p].variants[0].name+'&lt;/p>&lt;p class="desc">'+data[p].header+'&lt;/p>&lt;/td>';
              html += '&lt;td>'+data[p].variants[0].price_inc_vat_label+'&lt;/td>';
              html += '&lt;td nowrap="nowrap" class="produit_qte">&lt;input type="text" class="textfield" value="1" style="width:30px" id="qt-'+data[p].variants[0].id+'"/>&lt;/td>';                  
              html += '&lt;td colspan="2">&lt;input type="submit" style="margin-top:0px;" class="addsuggest" data-id="'+data[p].variants[0].id+'" value="Ajouter au panier"/>';
              html += '&lt;/td>&lt;/tr>';
              $('#p-'+this.promise.vid).after(html);
              break;                
            }
          }
        });
      }
      $(document).on('click', '.addsuggest', function(){
        kiubi.cart.addItem($(this).data('id'), $('#qt-'+$(this).data('id')).val()).done(function(){document.location.reload();});
      });
    });
  });
  &lt;/script>
  &lt;form method="post" action="">
    &lt;table class="table_commande">
      &lt;tr>
        &lt;th colspan="2" scope="col" style="width: 100%; text-align: left;">{nb_produits} article{pluriel_produits}&lt;/th>
        &lt;th nowrap="nowrap" scope="col">Prix&lt;/th>
        &lt;th scope="col" style="text-align: center;">Qte&lt;/th>
        &lt;th nowrap="nowrap" scope="col">Total&lt;/th>
        &lt;th scope="col" class="produit_supp">&nbsp;&lt;/th>
      &lt;/tr>
      &lt;!-- BEGIN:produit -->
      &lt;tr id="{ref}">
        &lt;td style="width: {miniature_l}px;" class="produit_illustration">&lt;!-- BEGIN:illustration -->
          &lt;figure>&lt;a href="{lien_produit}" title="{intitule_produit|htmlentities} - Cliquez pour acc&eacute;der">&lt;img src="{racine}/media/miniature/{illustration}" alt="{intitule_produit|htmlentities}" class="illustration" />&lt;/a>&lt;/figure>
          &lt;!-- END:illustration -->
          &lt;!-- BEGIN: noillustration -->
          &lt;figure>&lt;a href="{lien_produit}" title="{intitule_produit|htmlentities} - Cliquez pour acc&eacute;der">&lt;img src="{racine}/{theme}/fr/images/produit_mini.gif" alt="{intitule_produit|htmlentities}" class="illustration" style="width: {miniature_l}px; height: {miniature_h}px;" />&lt;/a>&lt;/figure>
          &lt;!-- END: noillustration -->&lt;/td>
        &lt;td class="produit_info">&lt;strong>&lt;a href="{lien_produit}">{intitule_produit}&lt;/a>&lt;/strong>&lt;br />
          &lt;!-- BEGIN:reference -->
          &lt;p class="ref">Ref. : {reference}&lt;/p>
          &lt;!-- END:reference -->
          &lt;p class="variante">{intitule_variante}&lt;/p>
          &lt;!-- BEGIN:accroche -->
          &lt;p class="desc">{accroche}&lt;/p>
          &lt;!-- END:accroche -->&lt;/td>
        &lt;td align="right" class="produit_pu">{prix_unitaire}&lt;/td>
        &lt;td nowrap="nowrap" class="produit_qte">&lt;input name="qt[{ref}]" type="text" class="textfield" value="{qt}" style="width:30px" />
          &lt;!-- BEGIN:erreur_qt -->
          &lt;div class="erreurs">max. : {max}&lt;/div>
          &lt;!-- END:erreur_qt -->&lt;/td>
        &lt;td align="right" class="produit_tt">{prix_total}&lt;/td>
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:produit -->
      &lt;!-- BEGIN:option -->
      &lt;tr>
    &lt;!-- BEGIN:simple -->
        &lt;td colspan="2" style="text-align: left;">{intitule_option} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
    &lt;!-- BEGIN:offerte -->
    &lt;td>&nbsp;&lt;/td>
    &lt;td style="text-align: center;">{qt}&lt;/td>
    &lt;td>Offert&lt;/td>
    &lt;!-- END:offerte -->
    &lt;!-- BEGIN:payante -->
    &lt;td>{prix_unitaire}&lt;/td>
    &lt;td style="text-align: center;">{qt}&lt;/td>
        &lt;td>{prix_total}&lt;/td>
    &lt;!-- END:payante -->
    &lt;!-- END:simple -->
    &lt;!-- BEGIN:textarea -->
        &lt;td colspan="2" style="text-align: left;">{intitule_option} : {valeur_option} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
    &lt;!-- BEGIN:offerte -->
    &lt;td>&nbsp;&lt;/td>
    &lt;td style="text-align: center;">{qt}&lt;/td>
    &lt;td>Offert&lt;/td>
    &lt;!-- END:offerte -->
    &lt;!-- BEGIN:payante -->
    &lt;td>{prix_unitaire}&lt;/td>
    &lt;td style="text-align: center;">{qt}&lt;/td>
        &lt;td>{prix_total}&lt;/td>
    &lt;!-- END:payante -->
    &lt;!-- END:textarea -->
    &lt;!-- BEGIN:select -->
        &lt;td colspan="2" style="text-align: left;">{intitule_option} : {valeur_option} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
    &lt;!-- BEGIN:offerte -->
    &lt;td>&nbsp;&lt;/td>
    &lt;td style="text-align: center;">{qt}&lt;/td>
    &lt;td>Offert&lt;/td>
    &lt;!-- END:offerte -->
    &lt;!-- BEGIN:payante -->
    &lt;td>{prix_unitaire}&lt;/td>
    &lt;td style="text-align: center;">{qt}&lt;/td>
        &lt;td>{prix_total}&lt;/td>
    &lt;!-- END:payante -->
    &lt;!-- END:select -->
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:option -->
      &lt;!-- BEGIN:bon -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">{intitule_produit} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
        &lt;td class="produit_tt">{prix_total}&lt;/td>
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:bon -->
      &lt;!-- BEGIN:remise -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">{intitule_remise} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
        &lt;td align="right">&nbsp;&lt;/td>
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:remise -->
      &lt;tr class="tva">
        &lt;td colspan="4" style="text-align: left;">&lt;strong>Total hors frais de livraison&lt;/strong>&lt;/td>
        &lt;td align="right">&lt;strong>{total_articles}&lt;/strong>&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- BEGIN:livrable -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">Frais de port applicables pour les livraisons vers : {zone_livraison}&lt;/td>
        &lt;td align="right">{frais_port}&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- END:livrable -->
      &lt;!-- BEGIN:nonlivrable -->
      &lt;tr class="tva">
        &lt;td colspan="4" style="text-align: left;">La commande d&eacute;passe le poids autoris&eacute; pour une livraison vers : {zone_livraison}&lt;/td>
        &lt;td align="right">Non livrable&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- END:nonlivrable -->
      &lt;!-- BEGIN:commande_HT -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">&lt;strong>Total HT&lt;/strong>&lt;/td>
        &lt;td align="right">&lt;strong>{total_HT}&lt;/strong>&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">TVA&lt;/td>
        &lt;td align="right">{total_TVA}&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- END:commande_HT -->
      &lt;tr class="ttc">
        &lt;td colspan="5" class="total" style="text-align: left;">&lt;span style="float: right;">{total_TTC}&lt;/span> Total TTC&lt;/td>
        &lt;td class="total produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
    &lt;/table>
    &lt;!-- BEGIN:options_disponibles -->
    &lt;div class="option_commande">
      &lt;h2>Envie d'une petite option ?&lt;/h2>
      &lt;table>
        &lt;!-- BEGIN:option -->
        &lt;tr class="{type_option}">
          &lt;!-- BEGIN:simple -->
          &lt;td>&lt;label>
              &lt;input name="options[]" value="{ref_option}" type="checkbox"/>
              {intitule_option}&lt;/label>
            &lt;div class="option_desc">{description_option}&lt;/div>&lt;/td>
          &lt;!-- BEGIN:offerte -->
          &lt;td>Offert&lt;/td>
          &lt;!-- END:offerte -->
          &lt;!-- BEGIN:payante -->
          &lt;td>{prix_option}&lt;/td>
          &lt;!-- END:payante -->
          &lt;!-- END:simple -->
          &lt;!-- BEGIN:textarea -->
          &lt;td>&lt;label>
              &lt;input name="options[]" value="{ref_option}" type="checkbox"/>
              {intitule_option}&lt;/label>
            &lt;div class="option_desc">{description_option}&lt;/div>
            &lt;textarea name="options_valeurs[{ref_option}]" rows="5">{valeur_option}&lt;/textarea>&lt;/td>
          &lt;!-- BEGIN:offerte -->
          &lt;td>Offert&lt;/td>
          &lt;!-- END:offerte -->
          &lt;!-- BEGIN:payante -->
          &lt;td>{prix_option}&lt;/td>
          &lt;!-- END:payante -->
          &lt;!-- END:textarea -->
          &lt;!-- BEGIN:select -->
          &lt;td>&lt;label>
              &lt;input name="options[]" value="{ref_option}" type="checkbox"/>
              {intitule_option}&lt;/label>
            &lt;div class="option_desc">{description_option}&lt;/div>
            &lt;select name="options_valeurs[{ref_option}]">
              &lt;!-- BEGIN:choix -->
              &lt;option value="{valeur}" {selected}>{valeur}&lt;/option>
              &lt;!-- END:choix -->
            &lt;/select>&lt;/td>
          &lt;!-- BEGIN:offerte -->
          &lt;td>Offert&lt;/td>
          &lt;!-- END:offerte -->
          &lt;!-- BEGIN:payante -->
          &lt;td>{prix_option}&lt;/td>
          &lt;!-- END:payante -->
          &lt;!-- END:select -->
          &lt;td class="option_supp">&nbsp;&lt;/td>
        &lt;/tr>
        &lt;!-- END:option -->
      &lt;/table>
    &lt;/div>
    &lt;!-- END:options_disponibles -->
    &lt;div class="bon_reduction">
      &lt;h2> Vous b&eacute;n&eacute;ficiez d'un bon de r&eacute;duction ? &lt;/h2>
      &lt;!-- BEGIN:erreur_bon -->
      &lt;div class="erreurs">{erreur}&lt;/div>
      &lt;!-- END:erreur_bon -->
      &lt;label for="reduction">Renseignez le code de votre bon de r&eacute;duction :&lt;/label>
      &lt;input type="text" name="bon" value="{bon}" class="textfield" id="reduction" style="width: 40%;"  />
    &lt;/div>
    &lt;p>
      &lt;input type="submit" name="bt_commande" value="Commander" class="submit" title="Commander"/>
      &lt;input type="submit" name="bt_recalcul" value="Recalculer" class="submit reset" title="Recalculer"/>
    &lt;/p>
    &lt;input type="hidden" name="act" value="refresh" />
    &lt;input type="hidden" name="ctl" value="{ctl}" />
    &lt;p class="a_right">&lt;a href="{lien_continuer}" class="bt_achats" title="Continuez vos achats">Continuez vos achats&lt;/a>&lt;/p>
  &lt;/form>
  &lt;!-- END:produits -->
  &lt;!-- BEGIN:noproduits -->
  &lt;p class="panier_vide">Votre panier est vide. &lt;a href="/" title="Continuez vos achats">Retournez &agrave; l'accueil&lt;/a>&lt;/p>
  &lt;!-- END:noproduits -->
&lt;/article>
&lt;!-- END:main -->
</pre>

## Les derniers produits vus

Cette fonctionnalité se fait en deux étapes, la première consiste à mémoriser un produit vu, la deuxième consiste à afficher une liste de ces mêmes produits.

### Mise en place

Remplacez le fichier `theme/fr/produits/simple/index.html` de votre thème par celui de ce tutoriel et ajouter le fichier `theme/fr/includes/produits_vus.html` dans votre thème.

Dans la mise en page de votre page d'accueil, ajouter un widget `Fragment de template` configuré avec le template `produits_vus` dans la sidebar et enregistrez la mise en page.

Visitez la page d'un produit puis retournez sur la page d'accueil de votre site, vous avez dans la sidebar un bloc affichant ce produit. En visitant d'autres produits, la sidebar en page d'accueil se met à jour en fonction des derniers produits vus.

### Explications

Examinons en détail le code du template `produits/simple/index.html` :

A chaque page affichée, on va stocker l'identifiant du produit dans un cookie, on va limiter la taille de ce cookie aux 3 derniers identifiants et rendre le cookie lisible sur tout le site (c'est la seule modification apportée au template original du détail produit) : 

<pre lang="html">
&lt;script type="text/javascript">
$(function(){
  var c = $.cookie('pv');
  if(c == null) c = [];
  else c = c.split('-');
  if($.inArray('{produit_id}', c)==-1) {
    c.push({produit_id});
    if(c.length>3) {
      c = c.slice(-3);
    }
    $.cookie('pv', c.join('-'), {path:'/'});
  }
});
&lt;/script>
</pre>

Examinons en détail le code du template `includes/produits_vus.html` :

Nous récupérons notre cookie, s'il n'est pas vide, on récupère le détail produit de chaque élément du cookie via l'API puis on ajoute un bloc contenant son titre, son image principale et une courte description à notre encart dans la sidebar.

L'encart est masqué par défaut, et n'est affiché que si au moins un produit a déjà été vu.

<pre lang="html">
&lt;script type="text/javascript">
$(function(){
  var c = $.cookie('pv');
  if(c !== null) {
    c = c.split('-');
    for(var i=0; i&lt;c.length; i++) {
      kiubi.catalog.getProduct(c[i], {extra_fields:'price_label'}).done(function(meta, data){
        var html = '&lt;div style="margin-bottom:15px;">';
        html += '&lt;h2>&lt;a href="'+data.url+'">'+data.name+'&lt;/a>&lt;/h2>';
        html += '&lt;a href="'+data.url+'">&lt;img src="'+data.main_thumb.url_miniature+'"/>&lt;/a>';
        html += '&lt;br/>Prix : '+data.price_min_inc_vat_label;
        html += '&lt;/div>';       
        $("#recent-view").show().append(html);      
      });
    }
  }
}); 
&lt;/script>
&lt;article class="block" id="recent-view" style="display:none;">&lt;h1>Derniers produits vus&lt;/h1>&lt;/article>
</pre>