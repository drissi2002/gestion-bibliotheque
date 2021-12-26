//Régle d'integrité

 --l’année d’édition (anEd) doit être après 1950

CREATE OR REPLACE TRIGGER   TrigAnneeSup1950Insertion
BEFORE INSERT ON catalogue 
FOR EACH ROW 
BEGIN
IF :NEW.anEd < 1950 THEN 
RAISE_APPLICATION_ERROR( -20118, ' l’année d’édition (anEd) doit être après 1950' ) ;
END IF ;
END;

------

 --On ne peut emprunter un titre qu’après avoir adhéré à la bibliothèque

CREATE OR REPLACE TRIGGER   TrigDateAdhesionInsertion
 BEFORE INSERT ON EMPRUNT 
 FOR EACH ROW 
 DECLARE 
 d_adh  DATE ; 
 BEGIN
 SELECT dateadh INTO d_adh 
 FROM adherent a  
 WHERE :NEW.noadh =a.noadh ; 
IF  :NEW.dateemp <= d_adh   THEN 
RAISE_APPLICATION_ERROR( -20105,  'On ne peut emprunter un titre qu’après avoir adhéré à la bibliothèque' ) ;
END IF ;
END;

------

 --Un emprunt ne doit pas dépasser un mois

CREATE OR REPLACE TRIGGER   TrigEmpruntMois
 BEFORE INSERT ON EMPRUNT 
 FOR EACH ROW 
 BEGIN     
IF  to_date(:NEW.daterprevue, 'dd-MM-yy') - to_date( :NEW.dateemp, 'dd-MM-yy') > 30 THEN 
RAISE_APPLICATION_ERROR( -20115,  ' Un emprunt ne doit pas dépasser un mois ' ) ;
END IF ;
END;

------

 --




------------------------------------------------------------------------------
//ADHERENT

 -- verifier qu'une personne n'est pas adherent avant ajout 

create or replace function f_verifAdherent 
(ncinp adherent.ncin%TYPE) return boolean is
     
    isAdherent boolean := false ;
    nbre number ; -- avce ce compteur on va verifier si la personne 
                  -- qu'on va l'ajouter est déja un adherent ou non
                  -- si nbre > 0 la personne deja existe dans la table
                  -- si non la personne n'est pas un adherent 
    begin
       SELECT count(*) into nbre from adherent 
       where ncin= ncinp ;
       if nbre > 0 then isAdherent:= true; end if ;
       return isAdherent ;
end f_verifAdherent;

create or replace trigger trigAdherent
    before insert on adherent
    for each row
    when (new.ncin is not null)
    
    declare
       verifAdherent boolean;
       
    begin
       verifAdherent:=f_verifAdherent(:new.ncin);
       if (verifAdherent =true) then
       RAISE_APPLICATION_ERROR(-20100,'cette personne est deja un adherent !!') ;
End if ;
End ;

--------------------------------------------------------------------------

 -- suppression impossible si l'adhrent a fait des emprunt

create or replace trigger trigSuppAdherent
    before delete on adherent
    for each row
    when (old.dateadh is not null)
     
    begin
       RAISE_APPLICATION_ERROR(-20101,'La suppression est impossible si l’adhérent a fait des emprunts !!') ;
    End ;

---------------------------------------------------------------------------

 -- Seulement l’adresse, le numéro de téléphone et l’email peuvent être modifiés.

SET SERVEROUTPUT ON
create or replace trigger trigModifAdherent
    before update of noadh , nom , prenom, ncin , dateadh   on adherent
    for each row 
    when(old.adresse=new.adresse and old.tel=new.tel and old.email=new.email )
    begin
       RAISE_APPLICATION_ERROR(-20102,'Seulement l’adresse, le numéro de téléphone et l’email peuvent être modifiés !') ;
    End ;

----------------------------------------------------------------------------

 -- La recherche ne se fait qu’avec le N°CIN.





----------------------------------------------------------------------------

//EMPRUNT 

 -- Refuser cette insertion et afficher une erreur si le nombre des emprunts qui ne sont pas encore rendus par 
l’adhérant est déjà égale à 5.  

SET SERVEROUTPUT ON
create or replace function f_verifEmprunt
(noadhEmp emprunt.noadh%TYPE ) return boolean is
     
    isEmprunt boolean := false ;
    nbre number ; -- avce ce compteur on va verifier si la personne 
                  -- qu'on va l'ajouter à deja emprunté plus que 5 
                  -- catalogue qui ne sont pas encore rendu
    begin
       select count(*)into nbre from emprunt 
       where ((noadh = noadhEmp) and ( DATERPREVUE > to_date(sysdate)));    
       if nbre >=5   then isEmprunt:= true; end if ;
       return isEmprunt ;
end f_verifEmprunt;

create or replace trigger trigRefInsertion
    before INSERT on emprunt
    for each row
    when (new.noadh is not null)
    
    declare
       verifEmprunt boolean;
    begin
       verifEmprunt:=f_verifEmprunt(:new.noadh);
       if (verifEmprunt=true) 
       then RAISE_APPLICATION_ERROR(-20104,'le nombre des emprunts qui ne sont pas encore rendus par 
       l’adhérant est déjà égale à 5!');
       end if;
    End ;                   (methode 1)


__________________

create or replace trigger trigRefInsertion
    before INSERT on emprunt
    for each row 
    declare
    nbre number ; -- compteur pour verifier si l'adherent a dépassé
                  -- le nombre des emprunts légale = 5
    begin
       select count(*)into nbre from emprunt 
       where ((noadh = :new.noadh) and ( DATERPREVUE > to_date(sysdate)));
       if (nbre >= 5) 
       then RAISE_APPLICATION_ERROR(-20104,'le nombre des emprunts qui ne sont pas encore rendus par 
       l’adhérant est déjà égale à 5!');
       end if;
    End ;                   (methode 2)


---------------------------------------------------------------------------

 -- Refuser cette insertion et afficher une erreur si cet adhérent a emprunté (pas encore rendu) un exemplaire 
du même ouvrage.


create or replace trigger trigMemeOuvrageInsertion
    before INSERT on emprunt
    for each row 
    declare
    nbre number ; -- si nbre =1 l'adherent deja a emprunté
                  -- une copie d'ou refuser l'insertion
    begin
       select count(*)into nbre from emprunt 
       where ((codexp = :new.codexp) and ( DATERPREVUE > to_date(sysdate)));
       if (nbre >= 5) 
       then RAISE_APPLICATION_ERROR(-20105,'cet adherent a deja emprunté un e copie !');
       end if;
    End ;   

---------------------------------------------------------------------------

 -- La valeur de la date de retour effective ne doit pas être antérieure à la date système.

create or replace trigger trigMemeOuvrageInsertion
    before INSERT on emprunt
    for each row 
    declare
    nbre number ; -- si nbre =1 l'adherent deja a emprunté
                  -- une copie d'ou refuser l'insertion
    begin
       select count(*)into nbre from emprunt 
       where ((codexp = :new.codexp) and ( DATERPREVUE > to_date(sysdate)));
       if (nbre >= 5) 
       then RAISE_APPLICATION_ERROR(-20105,'cet adherent a deja emprunté un e copie !');
       end if;
    End ; 




---------------------------------------------------------------------------

 -- Refuser cette insertion si l’adhérant a une pénalité de retard en cours. 

set serveroutput on
create or replace function f_verifAdherentRetard
(noadhRetard emprunt.noadh%TYPE ) return boolean is
     
    isRetard boolean := false ;
    nbre number ; -- avce ce compteur on va verifier si la personne 
                  -- à au moins une penalité de retard
                  -- si nbre >= 1 on va rejter l'insertion
    begin
       select count(*)into nbre from emprunt 
       where ((noadh = noadhRetard) and ( DATERPREVUE < datereffective ));    
       if nbre >=1    then isRetard:= true; end if ;
       return isRetard ;
end f_verifAdherentRetard;

create or replace trigger trigRetardInsertion
    before INSERT on emprunt
    for each row 
    when(new.noadh is not null)
    
    declare
       verifRetard boolean;
    begin
       verifRetard:=f_verifAdherentRetard(:new.noadh);
       if (verifRetard=true) 
       then RAISE_APPLICATION_ERROR(-20108,'Insertion refusé , penalité de retard existe !');
       end if;
    End ;  

---------------------------------------------------------------------------

 --le jour de retour prévue doit être toujours diffèrent de 29 et de 30.

set serveroutput on

create or replace trigger trigRetourInsertion 
before insert on emprunt 
for each row 

Begin
 if ( (to_date(:new.daterprevue) LIKE '29') or ( to_date(:new.daterprevue) LIKE '30')) 
   then RAISE_APPLICATION_ERROR(-20111, 'jour de retour ne peut pas etre 29 ou 30');
 end if;  
End;


---------------------------------------------------------------------------

 --Aucune suppression n’est acceptée.

create or replace trigger trigRefuséSuppression
before delete on emprunt 
for each row 

Begin
    RAISE_APPLICATION_ERROR(-20112 , 'Aucune suppression n’est acceptée');
End;

---------------------------------------------------------------------------

 -- Seul l’attribut dateReffective peut être modifié mais pas quand il y’a un retard. (dateReffective> 
dateRprevue).

set serveroutput on

SET SERVEROUTPUT ON
create or replace trigger trigModifEmprunt
    before update of codexp ,dateemp , noadh,daterprevue  on emprunt
    for each row 
    when(old.datereffective=new.datereffective)
    begin
    if  (:old.datereffective > :old.daterprevue)
    then RAISE_APPLICATION_ERROR(-20102,'dateReffective peut être modifié mais pas quand il y’a un retard!');
    end if;
    End ;
----------------------------------------------------------------------------

 -- La recherche peut se faire soit:
                  - avec le codeg,
                  -Sur une periode, 
                  - avec noAdh
                  - avec noAdh et le codexp  (ces 4 procedure deja definies dans le package
                                              gestionEmprunts . )
---------------------------------------------------------------------------------

//Package Adherent 

create or replace package gestionAdherents as
function f_chercherAdherent (ncinAdh IN adherent.ncin%type )
return  number ;
function f_verifAdherent (ncinAdh IN adherent.ncin%TYPE)
return  boolean ;
procedure p_ajouterAdherent (nomAdh IN adherent.nom%type , prenomAdh IN adherent.prenom%type , adresseAdh IN adherent.adresse%type ,ncinAdh IN adherent.ncin%type , telAdh IN adherent.tel%type , emailAdh adherent.email%type); 
procedure p_updateAdherent(cinAdh IN adherent.ncin%type , adresseAdh IN adherent.adresse%type , emailAdh IN adherent.email%type , telAdh IN adherent.tel%type);
PROCEDURE p_rechercheAdhCin (cinadh IN adherent.ncin%TYPE);
End  ;

create or replace package body gestionAdherents as

function f_chercherAdherent (ncinAdh IN adherent.ncin%type )
return number  is
nbre number;
begin 
select count(*) into nbre from adherent where ncin=ncinAdh;
return nbre;
END f_chercherAdherent;

function f_verifAdherent (ncinAdh IN adherent.ncin%TYPE) 
return boolean is

    isAdherent boolean := false ;
    nbre number ; -- avce ce compteur on va verifier si la personne 
                  -- qu'on va l'ajouter est déja un adherent ou non
                  -- si nbre > 0 la personne deja existe dans la table
                  -- si non la personne n'est pas un adherent 
    begin
       SELECT count(*) into nbre from adherent 
       where ncin= ncinAdh ;
       if nbre > 0 then isAdherent:= true; end if ;
       return isAdherent ;
end f_verifAdherent;

procedure p_ajouterAdherent (nomAdh IN adherent.nom%type , prenomAdh IN adherent.prenom%type , adresseAdh IN adherent.adresse%type ,ncinAdh IN adherent.ncin%type , telAdh IN adherent.tel%type , emailAdh adherent.email%type)is 
begin
INSERT INTO adherent(noadh,nom,prenom,adresse,ncin,tel,dateadh,email) VALUES ( noAdherent_seq.nextval,nomAdh,prenomAdh,adresseAdh,ncinAdh,telAdh, to_date(sysdate),emailAdh );
end p_ajouterAdherent;

procedure p_updateAdherent(cinAdh IN adherent.ncin%type , adresseAdh IN adherent.adresse%type , emailAdh IN adherent.email%type , telAdh IN adherent.tel%type) is 
begin 
update adherent set adresse = adresseAdh ,email=emailAdh , tel=telAdh 
where ncin=cinAdh;
END p_updateAdherent;


PROCEDURE p_rechercheAdhCin (cinadh IN adherent.ncin%TYPE) is
CURSOR c2 IS SELECT noadh ,nom,prenom,adresse,ncin,tel,dateadh,email
FROM adherent
WHERE ncin=cinadh ; 
vadh c2%ROWTYPE;
ERROR_c2 exception ; 
BEGIN 
OPEN c2 ;
LOOP
FETCH c2 INTO vadh;
EXIT WHEN c2%NOTFOUND OR c2%NOTFOUND IS NULL ;
DBMS_OUTPUT.PUT_LINE(' le cin d adherent est  ' || vadh.ncin || '  son non est   '|| vadh.nom|| ' son prenom est   '||vadh.prenom );
dbms_output.new_line(); 
END LOOP; 
IF c2%ROWCOUNT=0 THEN RAISE ERROR_c2 ; 
END IF ; 
CLOSE c2 ; 
EXCEPTION
WHEN  ERROR_c2 then dbms_output.put_line('il n y a pas adherent avec cette CIN  ');
END p_rechercheAdhCin ;
End gestionAdherents ;

--------------------------------------------------------------------------------

//Package Emprunt

create or replace package gestionEmprunts as
function f_chercherEmprunt (noadhEmp IN emprunt.noadh%TYPE )
return  number ;
function f_verifEmprunt(noadhEmp IN emprunt.noadh%TYPE )
return  boolean ;
PROCEDURE p_rechercheCodeg (code IN emprunt.codexp%type);
PROCEDURE p_recherchePeriode (date1 IN emprunt.dateEmp%TYPE,date2 IN emprunt.dateRprevue%TYPE);
PROCEDURE p_rechercheAdh (no_adh IN emprunt.noadh%TYPE);
 PROCEDURE p_rechercheAdhExp(no_adh emprunt.noadh%TYPE,exp emprunt.codexp%TYPE);
End  ;

create or replace package body gestionEmprunts as

function f_chercherEmprunt (noadhEmp IN emprunt.noadh%TYPE )
return number  is
nbre number;
begin 
select count(*) into nbre from Emprunt where noadh=noadhEmp;
return nbre;
END f_chercherEmprunt;

function f_verifEmprunt(noadhEmp IN emprunt.noadh%TYPE )
return boolean is

    isEmprunt boolean := false ;
    nbre number ; -- avce ce compteur on va verifier si la personne 
                  -- qu'on va l'ajouter à deja emprunté plus que 5 
                  -- catalogue qui ne sont pas encore rendu
    begin
       select count(*)into nbre from emprunt 
       where ((noadh = noadhEmp) and ( DATERPREVUE > to_date(sysdate)));    
       if nbre >=5   then isEmprunt:= true; end if ;
       return isEmprunt ;
end f_verifEmprunt;
--codeg
PROCEDURE p_rechercheCodeg (code IN emprunt.codexp%type) is
CURSOR c IS SELECT emp.codexp ,emp.dateemp,emp.datereffective,emp.daterprevue,emp.noadh 
FROM emprunt emp,exemplaire exp
WHERE exp.codexp=emp.codexp
AND exp.codeg=code;
vcode c%ROWTYPE;
ERROR_C exception ; 
BEGIN 
OPEN c ;
LOOP
FETCH c INTO vcode;
EXIT WHEN c%NOTFOUND OR c%NOTFOUND IS NULL ;
DBMS_OUTPUT.PUT_LINE(' le code exemplaire est  ' || vcode.codexp || '  la date d''emprunt  est   '|| vcode.dateemp|| '   la date de retour effective  est   '|| vcode.datereffective|| '   la date de retour prevue  est   '||vcode.daterprevue||'   la date de retour effective  est   '|| vcode.datereffective || '    le numero de l''adherent   est   '||vcode.noadh );
dbms_output.new_line(); 
END LOOP; 
IF c%ROWCOUNT=0 THEN RAISE ERROR_C ; 
END IF ; 
CLOSE c ; 
EXCEPTION
WHEN  ERROR_C then dbms_output.put_line('il n y a pas d"emprunt avec ce code ');
END p_rechercheCodeg;


--periode
PROCEDURE p_recherchePeriode (date1 IN emprunt.dateEmp%TYPE,date2 IN emprunt.dateRprevue%TYPE) is
CURSOR c1 IS SELECT emp.codexp ,emp.dateemp,emp.datereffective,emp.daterprevue,emp.noadh 
FROM emprunt emp
WHERE emp.dateEmp=date1
AND emp.dateRprevue=date2;
vperoide c1%ROWTYPE;
ERROR_C1 exception ; 
BEGIN 
OPEN c1 ;
LOOP
FETCH c1 INTO vperoide;
EXIT WHEN c1%NOTFOUND OR c1%NOTFOUND IS NULL ;
DBMS_OUTPUT.PUT_LINE(' le code exemplaire est  ' || vperoide.codexp || '  la date d''emprunt  est   '|| vperoide.dateemp|| ' le numero de l''adherent   est   '||vperoide.noadh||   '  la date de retour prevue  est   '||vperoide.daterprevue||'   la date de retour effective  est   '|| vperoide.datereffective  );
dbms_output.new_line(); 
END LOOP; 
IF c1%ROWCOUNT=0 THEN RAISE ERROR_C1 ; 
END IF ; 
CLOSE c1 ; 
EXCEPTION
WHEN  ERROR_C1 then dbms_output.put_line('il n y a pas d"emprunts dans cette periode ');
END p_recherchePeriode;



--noadh
PROCEDURE p_rechercheAdh (no_adh IN emprunt.noadh%TYPE) is
CURSOR c2 IS SELECT codexp ,dateemp,datereffective,daterprevue,noadh 
FROM emprunt
WHERE noadh=no_adh ; 
vadh c2%ROWTYPE;
ERROR_c2 exception ; 
BEGIN 
OPEN c2 ;
LOOP
FETCH c2 INTO vadh;
EXIT WHEN c2%NOTFOUND OR c2%NOTFOUND IS NULL ;
DBMS_OUTPUT.PUT_LINE(' le code exemplaire est  ' || vadh.codexp || '  la date d''emprunt  est   '|| vadh.dateemp|| ' le numero de l''adherent   est   '||vadh.noadh||   '  la date de retour prevue  est   '||vadh.daterprevue||'   la date de retour effective  est   '|| vadh.datereffective  );
dbms_output.new_line(); 
END LOOP; 
IF c2%ROWCOUNT=0 THEN RAISE ERROR_c2 ; 
END IF ; 
CLOSE c2 ; 
EXCEPTION
WHEN  ERROR_c2 then dbms_output.put_line('il n y a pas d"emprunts pour cet adherent  ');
END p_rechercheAdh ;


-- noadh et codexp
PROCEDURE p_rechercheAdhExp(no_adh emprunt.noadh%TYPE,exp emprunt.codexp%TYPE) is
CURSOR c1 IS SELECT emp.codexp ,emp.dateemp,emp.datereffective,emp.daterprevue,emp.noadh 
FROM emprunt emp
WHERE emp.noadh=no_adh
AND emp.codexp=exp;
vexp c1%ROWTYPE;
ERROR_C1 exception ; 
BEGIN 
OPEN c1 ;
LOOP
FETCH c1 INTO vexp;
EXIT WHEN c1%NOTFOUND OR c1%NOTFOUND IS NULL ;
DBMS_OUTPUT.PUT_LINE(' le code exemplaire est  ' || vexp.codexp || '  la date d''emprunt  est   '|| vexp.dateemp|| ' le numero de l''adherent   est   '||vexp.noadh||   '  la date de retour prevue  est   '||vexp.daterprevue||'   la date de retour effective  est   '|| vexp.datereffective  );
dbms_output.new_line(); 
END LOOP; 
IF c1%ROWCOUNT=0 THEN RAISE ERROR_C1 ; 
END IF ; 
CLOSE c1 ; 
EXCEPTION
WHEN  ERROR_C1 then dbms_output.put_line('il n y a pas d"emprunts avec ces criteres ');
END p_rechercheAdhExp;
End gestionEmprunts ;
---------------------------------------------------------------------------------

-- Calcul de pénalité : nombredejours = dateReffective – dateRprevue

CREATE TABLE retard
( noadh number(6) PRIMARY KEY ,
  codeg number(10) NOT NULL,  
  dateEmp date NOT NULL ,
  dateReffective date NOT NULL,
  penalite number(10) ,
  encours VARCHAR(20)    
);

create or replace function diffDate (date1 date , date2 date) return number as 
   begin
   if (to_date(date1 ,'MM')- to_date(date2,'MM')=0)then
   return to_date(date1,'DD')-to_date(date2,'DD');
   else  
       return (to_date(date1 ,'MM')- to_date(date2,'MM'))*30 +(to_date(date1,'DD')-to_date(date2,'DD'));
   end if;
   end;
   
create or replace trigger retardTrigger 
    before update on Emprunt 
    for each row 
    declare 
    retardJour number ;
       catalogue exemplaire.codeg%type;
    begin 

    retardJour := diffDate(:new.dateReffective,:old.dateRprevue);
    select ex.codeg into  catalogue from exemplaire ex, emprunt e  where  (ex.codexp = e.codexp );
    if(retardJour <  30)then 
    insert into retard(noadh,codeg,dateemp,datereffective,penalite,encours)values(:new.noadh,catalogue,:new.dateemp,:new.datereffective,retardJour,'non');
    else if(retardJour > 30 and retardJour < 90) then
    insert into retard(noadh,codeg,dateemp,datereffective,penalite,encours)values(:new.noadh,catalogue,:new.dateemp,:new.datereffective,retardJour*2,'non');
    else if (retardJour > 90)then 
    DBMS_output.put_line('payer le prix d ouvrage');
    end if ;
    end if ;
 end if ;
 end retardTrigger;
   
---------------------------------------------------------------------------------

--L’attribut encours prend la valeur ‘oui’ à l’insertion d’un retard et la valeur ‘non’ après 
l’expiration de la durée de la pénalité. Proposer une solution pour la mise à jour de la table RETARD.

create or replace trigger retard_encours_trigger BEFORE update of encours on retard 
for each row 
begin
if(:new.datereffective + :new.penalite >to_date(sysdate))then
raise_application_error(-20100,'depassement des jours de penalite');
end;


---------------------------------------------------------------------------------

--Créer une vue VStatistiques2018 qui va chercher pour chaque ouvrage, le nombre d’exemplaires, le 
nombre d’exemplaires qui ne sont pas disponibles car ils sont dans un état médiocre, le nombre 
d’emprunt et le pourcentage de retours en retard.


--CREATE OR REPLACE VIEW VStatistiques2018 (codcatlogue, nbreExpTot,nbreExpINDISPO,nbreEmprunt,prcRetard)
--as 
select codeg from catalogue; 
select count(*) from exemplaire; 
select count(*)from exemplaire where etat like 'mediocre';
select count(*)from exemplaire , emprunt 
where emprunt.codexp=exemplaire.codexp ;
select count(*) from exemplaire , emprunt 
where emprunt.codexp=exemplaire.codexp and (emprunt.datereffective is null or emprunt.datereffective> emprunt.daterprevue);







---------------------------------------------------------------------------------

 -- On veut interdire l’accès aux tables catalogue, emprunt et exemplaire le 29 et le 30 de chaque mois.


create or replace TRIGGER AlerteMiseAjourCatalogue
BEFORE INSERT OR UPDATE OR DELETE ON catalogue 
BEGIN
IF (TO_CHAR(SYSDATE,'DD') ='29' or TO_CHAR(SYSDATE,'DD') ='30')then

Raise_application_error(-20102,'Désolé pas de manipulation des
commandes le 29 et le 30');
END if;
End;


----------

create or replace TRIGGER AlerteMiseAjourEmprunt
BEFORE INSERT OR UPDATE OR DELETE ON Emprunt 
BEGIN
IF (TO_CHAR(SYSDATE,'DD') ='29' or TO_CHAR(SYSDATE,'DD') ='30')then

Raise_application_error(-20102,'Désolé pas de manipulation des
commandes le 29 et le 30');
END if;
End;


-----------

create or replace TRIGGER AlerteMiseAjourExemplaire
BEFORE INSERT OR UPDATE OR DELETE ON Exemplaire 
BEGIN
IF (TO_CHAR(SYSDATE,'DD') ='29' or TO_CHAR(SYSDATE,'DD') ='30')then

Raise_application_error(-20102,'Désolé pas de manipulation des
commandes le 29 et le 30');
END if;
End;






