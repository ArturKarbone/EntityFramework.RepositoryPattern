EntityFramework.RepositoryPattern
=================================


public class BaseRepository<TEntity> : IBaseRepository<TEntity> where TEntity : class, IDBIdentity
    {
        Func<BaseContext, DbSet<TEntity>> ExtractDbSetFromContext { get; set; }

        public BaseRepository(Func<PPKCEntities, DbSet<TEntity>> extractDbSetFromContext)
        {
            this.ExtractDbSetFromContext = extractDbSetFromContext;
        }

        public virtual List<TEntity> GetAll()
        {
            using (var db = new BaseContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                return dbSet.ToList();
            }
        }

        public virtual TEntity Get(Guid id)
        {
            using (var db = new BaseContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                return dbSet.FirstOrDefault(x => x.Id == id);
            }
        }

        public virtual TEntity InserOrUpdate(TEntity entity)
        {
            using (var db = new BaseContext())
            {
                var dbSet = ExtractDbSetFromContext(db);

                db.Entry(entity).State = GetStateBasedOnIdentity(entity);

                EnsureIdentity(entity);

                entity.CreatedAt = DateTime.UtcNow;
                entity.UpdatedAt = DateTime.UtcNow;

                db.SaveChanges();

                return entity;
            }
        }

        private static void EnsureIdentity(TEntity entity)
        {
            if (entity.Id == Guid.Empty)
            {
                entity.Id = Guid.NewGuid();
            }
        }

        private EntityState GetStateBasedOnIdentity(TEntity entity)
        {
            return entity.Id == Guid.Empty || Get(entity.Id) == null ? EntityState.Added : EntityState.Modified;
        }

        public virtual void Delete(Guid id)
        {
            using (var db = new BaseContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                var entity = dbSet.FirstOrDefault(x => x.Id == id);
                if (null != entity)
                {
                    dbSet.Remove(entity);
                }
                db.SaveChanges();
            }
        }

        public virtual void Copy(Guid id)
        {
            using (var db = new BaseContext())
            {
                var dbSet = ExtractDbSetFromContext(db);
                var entityToCopy = dbSet.AsNoTracking().FirstOrDefault(x => x.Id == id);
                entityToCopy.Id = Guid.NewGuid();
                dbSet.Add(entityToCopy);
                db.SaveChanges();
            }
        }
    }
