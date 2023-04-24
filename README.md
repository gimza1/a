RANDOM:

CREATE OR REPLACE PACKAGE customer_pkg AS

  -- CREATE
  PROCEDURE create_customer (
    p_customer_id IN NUMBER,
    p_first_name IN VARCHAR2,
    p_last_name IN VARCHAR2,
    p_email IN VARCHAR2
  );
  
  -- READ
  FUNCTION get_customer (
    p_customer_id IN NUMBER
  ) RETURN SYS_REFCURSOR;
  
  -- UPDATE
  PROCEDURE update_customer (
    p_customer_id IN NUMBER,
    p_first_name IN VARCHAR2,
    p_last_name IN VARCHAR2,
    p_email IN VARCHAR2
  );
  
  -- DELETE
  PROCEDURE delete_customer (
    p_customer_id IN NUMBER
  );
  
END customer_pkg;
/

CREATE OR REPLACE PACKAGE BODY customer_pkg AS

  -- CREATE
  PROCEDURE create_customer (
    p_customer_id IN NUMBER,
    p_first_name IN VARCHAR2,
    p_last_name IN VARCHAR2,
    p_email IN VARCHAR2
  ) AS
  BEGIN
    INSERT INTO customers (
      customer_id,
      first_name,
      last_name,
      email
    ) VALUES (
      p_customer_id,
      p_first_name,
      p_last_name,
      p_email
    );
    COMMIT;
  END;
  
  -- READ
  FUNCTION get_customer (
    p_customer_id IN NUMBER
  ) RETURN SYS_REFCURSOR AS
    v_cursor SYS_REFCURSOR;
  BEGIN
    OPEN v_cursor FOR
      SELECT *
      FROM customers
      WHERE customer_id = p_customer_id;
    RETURN v_cursor;
  END;
  
  -- UPDATE
  PROCEDURE update_customer (
    p_customer_id IN NUMBER,
    p_first_name IN VARCHAR2,
    p_last_name IN VARCHAR2,
    p_email IN VARCHAR2
  ) AS
  BEGIN
    UPDATE customers
    SET first_name = p_first_name,
        last_name = p_last_name,
        email = p_email
    WHERE customer_id = p_customer_id;
    COMMIT;
  END;
  
  -- DELETE
  PROCEDURE delete_customer (
    p_customer_id IN NUMBER
  ) AS
  BEGIN
    DELETE FROM customers
    WHERE customer_id = p_customer_id;
    COMMIT;
  END;
  
END customer_pkg;
/

MUKI PROCEDURE:
CREATE OR REPLACE PROCEDURE get_customers(page_no NUMBER,
page_size NUMBER)
AS
c_customers SYS_REFCURSOR;
c_total_row SYS_REFCURSOR;
BEGIN
-- otvaranje prvog kursora ciji ce rezultujuci set podataka
-- biti vracen procedurom
OPEN c_total_row FOR
SELECT COUNT(*)
FROM customers;

DBMS_SQL.RETURN_RESULT(c_total_row);

-- otvaranje drugog kursora ciji ce rezultujuci set podataka
-- biti vracen procedurom
OPEN c_customers FOR
SELECT customer_id, name
FROM customers
ORDER BY name
OFFSET page_size * (page_no - 1) ROWS
FETCH NEXT page_size ROWS ONLY;

DBMS_SQL.RETURN_RESULT(c_customers);
END;

MUKI FUNCTION:
CREATE OR REPLACE FUNCTION get_total_sales(in_year PLS_INTEGER)
RETURN NUMBER
IS
l_total_sales NUMBER := 0;
BEGIN
-- izvrsavanje upita koji vraca informaciju o ukupnoj prodaji
-- u odredjenoj godini
SELECT SUM(unit_price * quantity)
INTO l_total_sales
FROM order_items
INNER JOIN orders USING (order_id)
WHERE status = 'Shipped'
GROUP BY EXTRACT(YEAR FROM order_date)
HAVING EXTRACT(YEAR FROM order_date) = in_year;

-- funkcija vraca rezultat dobijen upitom
RETURN l_total_sales;
END;



MUKI

CREATE OR REPLACE NONEDITIONABLE PACKAGE BODY "BOJANMUVRIN"."PAC_DRZAVA" AS
  -- Implementacija funkcije koja vraća pokazivač na tabelu država regiona.
  FUNCTION fn_Kursor_Drzave(pa_Region_ID IN NUMBER,
                            pa_Country_ID IN VARCHAR2) RETURN SYS_REFCURSOR AS
  
    -- Tekst upita za selektovanje svih zemalja regiona.
    l_Upit VARCHAR2(200) := 'SELECT * FROM Countries ' ||
                             'WHERE Region_ID = ' || pa_Region_ID;
    c_Drzave SYS_REFCURSOR;
  BEGIN
    -- Ako je prosledjen i ID konkretne države,
    if pa_Country_ID IS NOT NULL THEN
      l_Upit := l_Upit || q'[ AND Country_ID = ']' || pa_Country_ID || q'[']';
    END IF;
    
    DBMS_OUTPUT.PUT_LINE(l_Upit);
    
    -- Otvaranje kursora sa podacima tabele Countries za prosleđeni region
    -- i/ili konkretnu državu.
    OPEN c_Drzave FOR l_Upit;
    
    -- Vraćanje pokazivača na rezultujuću tabelu.
    RETURN c_Drzave;
  END fn_Kursor_Drzave;

  /* Implementacija procedure za dodavanje novog sloga ili promenu podataka u
   postojećem slogu.
   
   INSERT operacija :
     vrednost ulaznih parametara : 
       pa_Country_ID - "" (prazan string),
       pa_Country_Name - ime nove države
       pa_Region_ID - ID regiona kojem pripada nova država.
     vrednost izlaznog parametra :
       pa_DodeljenIDPriINSERT - dodeljena vrednost za Country_ID od
                                strane sistema.
                                
   UPDATE operacija : 
     vrednost ulaznih parametara : 
       pa_Country_ID - vrednost ID-ja države čiji se podaci menjaju,
       pa_Country_Name - novo (promenjeno) ime države,
       pa_Region_ID - ID regiona kojem pripada država.
     vrednost izlaznog parametra :
       pa_DodeljenIDPriINSERT - ne dodeljuje se.     
  */
  PROCEDURE pcd_Drzava_Insert_Update(pa_Country_ID IN VARCHAR2,
                                     pa_Country_Name IN VARCHAR2,
                                     pa_Region_ID IN NUMBER,
                                     pa_DodeljenIDPriINSERT OUT VARCHAR2) AS
    -- Naziv države je jedinstven u tabeli. 
    ex_DuplikatNazivaDrzave EXCEPTION;
    PRAGMA EXCEPTION_INIT(ex_DuplikatNazivaDrzave, -20001);

    -- Deklaracija kursora za UPDATE.
    CURSOR c_DrzaveUPDATE IS 
    SELECT 
      Country_Name
    FROM 
      Countries
    WHERE 
      Country_ID = pa_Country_ID
    FOR UPDATE OF Country_Name;
  BEGIN
    -- Ako je operacija UPDATE :
    IF pa_Country_ID <> '' THEN
      -- Ako postoji država sa istim imenom, 
      IF fn_Broj_Slogova_U_Tabeli(q'[Countries WHERE UPPER(Country_Name)=']' ||
                                  UPPER(pa_Country_Name) ||
                                  q'[' AND Country_ID <> ']' || 
                                  pa_Country_ID || q'[']')>0  THEN
        -- pokreće se izuzetak.
        RAISE ex_DuplikatNazivaDrzave;
      END IF;  
    
      -- U suprotnom, otvara se kursor za UPDATE
      FOR r_Drzava IN c_DrzaveUPDATE
      LOOP
        -- i UPDATE-uje se odgovarajući slog tabele.
        UPDATE Countries SET Country_name = pa_Country_Name
        WHERE CURRENT OF c_DrzaveUPDATE;
      END LOOP; 
    
      COMMIT;
    -- U suprotnom, ako je operacija INSERT,  
    ELSE
      -- Ako postoji država sa istim imenom,     
      IF fn_Broj_Slogova_U_Tabeli(q'[Countries WHERE UPPER(Country_Name)=']' ||
                                  UPPER(pa_Country_Name) || q'[']')>0 THEN
        -- pokreće se izuzetak.
        RAISE ex_DuplikatNazivaDrzave;
      END IF;  

      -- U suprotnom, dodavanje novog sloga u tabelu sa vraćanjem vrednosti
      -- generisanog Country_ID.
      INSERT INTO Countries (Country_ID, Country_Name, Region_ID)
      VALUES (UPPER(SUBSTR(pa_Country_Name, 1, 2)),
              pa_Country_Name,
              pa_Region_ID)
      RETURNING Country_ID INTO pa_DodeljenIDPriINSERT;
    
      COMMIT;
    END IF;
  END pcd_Drzava_Insert_Update;

  -- Implementacija funkcije za proveru postojanja zavisnih tabela u 
  -- odnosu na državu koja se briše.
  FUNCTION fn_Broj_Slogova_U_Zavisnim_Tabelama_Drzave(pa_Country_ID IN VARCHAR2)
    RETURN SYS_REFCURSOR AS
    c_Tabela_Broja_Slogova_U_Zavisnim_Tabelama_Drzave SYS_REFCURSOR;  
  BEGIN
    -- Otvaranje kursora sa podacima o broju slogova u zavisnim tabelama.
    OPEN c_Tabela_Broja_Slogova_U_Zavisnim_Tabelama_Drzave FOR
      SELECT COUNT(DISTINCT Location_ID) AS BrojLokacija,
             COUNT(DISTINCT Warehouse_ID) AS BrojMagacina,
             COUNT(DISTINCT Product_ID) AS BrojProizvodaNaLageruUMagacinima
      FROM Locations LEFT JOIN Warehouses USING (Location_ID)
                     LEFT JOIN Inventories USING (Warehouse_ID)
      WHERE Locations.Country_ID = pa_Country_ID;
    
    -- Vraćanje pokazivača na rezultujuću tabelu.
    RETURN c_Tabela_Broja_Slogova_U_Zavisnim_Tabelama_Drzave;
    
  END fn_Broj_Slogova_U_Zavisnim_Tabelama_Drzave;

  -- Implementacija procedure za brisanje države.
  PROCEDURE pcd_Brisanje_Drzave(pa_Country_Id IN VARCHAR2) AS
  BEGIN
    DELETE
    FROM Countries
    WHERE Country_ID = pa_Country_ID;

    COMMIT;
  END pcd_Brisanje_Drzave;

END pac_Drzava;
